# Decisions

Architecture Decision Records. Every non-obvious choice, with the rationale so a future AI doesn't undo it without understanding why.

---

## ADR-001: File → Summary → Index is the spine

**Decision:** The core output shape is three layers — per-file forensic analysis, per-group `_SUMMARY.md`, and repo-level `INDEX.md` — with a `manifest.json` dependency graph underneath.

**Why:** Every other shape — database, web UI, search index, property graph — is a view on top of this. Plain markdown + JSON is universal, diffable, git-friendly, and human-readable. Any future tool can read it.

**Alternatives considered:**
- Single big document (too slow to update incrementally)
- Database-backed (adds ops burden, loses git-friendliness)
- AST-based graph only (loses the prose reasoning that LLMs produce well)

---

## ADR-002: dbt-style manifest for staleness

**Decision:** `manifest.json` stores a sha256 prefix of every source file at analysis time. `sync` diffs current SHAs against stored SHAs to find stale nodes, walks `depended_on_by` for the closure.

**Why:** It's exactly how dbt handles `state:modified+` and it works. The alternatives (mtime-based, git-diff-based) are less robust — mtime lies across machines, git-diff requires a clean working tree.

**Invariant:** Never trust mtime alone. Always recompute SHA.

---

## ADR-003: LLM delegation via inline-prompt + stdin (was: slash command)

**Decision (current — 2026-04-08):** LLM-powered CLI commands read `.claude/commands/docs-*.md` from disk as **prompt source**, render `{{VARS}}` in Node, and pipe the inlined body to `claude -p` over **stdin**. The CLI is still a thin shim, but it no longer invokes a slash command.

**Why the change:**
1. **Claude Code 2.1.92 does not execute slash commands when invoked via `-p`.** `claude -p "/docs-init"` was a no-op: the model received the literal text "/docs-init" and either echoed `Unknown command` or wandered off. There is no way to make `-p` resolve a project-local slash command.
2. **Windows argv mangles CRLF inside long prompt bodies.** Even when slash invocation worked elsewhere, multi-kilobyte prompts passed as a single argv string lost line endings on Windows. Stdin sidesteps both problems entirely.
3. **It removes the "did the user copy the .md files into `~/.claude/commands/`?" failure mode.** The runner reads the project-local files directly. There is no install step.

**The .md files in `.claude/commands/` are still authoritative** — they remain the prompt source the runner reads. They are no longer "Claude Code slash commands" at runtime; they are prompt templates with `{{VARS}}` placeholders.

**Invariants preserved from the original ADR:**
- The deterministic-vs-LLM split is unchanged (ADR-004).
- The CLI is still a thin shim around prompts (ADR-014).
- `manifest.json` remains the single source of truth (ADR-015).
- `REPO_MIND_LLM_CMD` is still the swap point for non-Claude runners; an alternate runner just needs to accept a prompt body on stdin.

**New invariant added by this ADR:** every LLM-delegating command verifies its artifacts after the runner returns. Zero artifacts written, or `Unknown skill` / `Unknown command` leakage in stdout, fails the command loudly. `doctor` round-trips a sentinel through the real runner (`--skip-llm-probe` for offline use).

**Do not reintroduce `claude -p "/docs-..."` invocation without re-testing on Windows + Claude Code ≥ 2.1.92.**

**Original decision (historical — superseded above):** shell out to `claude -p "/docs-init"`. Failed in production for the reasons above.

---

## ADR-017: Branch-parity mode for multi-branch documentation (2026-04-08)

**Decision:** `docs.config.json` gains an optional `branches` block:

```json
"branches": {
  "mode": "single",
  "target_worktree_root": "../worktrees",
  "pin_model": "claude-opus-4-6",
  "require_clean_target": true
}
```

`mode: "single"` (default) preserves all existing behavior. `mode: "parity"` derives `target_repo` as `<target_worktree_root>/<docs-branch-name>` and enforces docs-branch-name === target-branch-name on every command.

New commands: `branch-status`, `branch-sync [--create]`, `ask "<q>" [--branch <name>]`, `resolve-merge <file>`, `watch --branch-mode`. New global `--ref <sha>` flag on every command — if target HEAD doesn't match `manifest.metadata.git_ref` and `--ref` doesn't reconcile them, the command refuses to run.

**Why:**
1. Real teams have multiple long-lived branches with diverging contracts. Forcing a single docs branch loses the ability to ask "what does AUDIT.md look like for the release branch?"
2. Driving target_repo from a worktree root keeps the docs store as the source of truth. The user creates one worktree per branch with `git worktree add`; the runner manages the docs side.
3. The default stays `single` so no existing user is forced to migrate.

**Invariants preserved:**
- No new state directory. `manifest.metadata.git_ref` (already part of spec v1.5) is the anchor.
- Manifest-is-authoritative (ADR-015) — the new commands all read/write the existing manifest fields.
- Deterministic-vs-LLM split (ADR-004) — `branch-status`, `branch-sync`, `resolve-merge` are pure Node; only `ask` may go through the runner.

**`resolve-merge`** is a git merge driver registered against analysis `.md` files. It uses each side's frontmatter `source_sha` to decide which branch's analysis is current. Install lines for `.gitattributes` and `.git/config` are documented in the cheatsheet but not auto-installed — the user opts in.

**`doctor`** warns when `target_repo` is inside the same git repo as the docs store and suggests a `git worktree add` layout to fix it.

**SPEC.md bumped to 1.8** (additive). The JSON schema `spec_version` field still reads `"1"`.

---

## ADR-004: Deterministic plane must never call an LLM

**Decision:** Commands like `status`, `validate`, `manifest`, `symbols`, `glossary`, `entry-points`, `dead-code`, `graph`, `intents`, `compliance`, `coverage` are **pure Node** and never call an LLM. Only `init`, `sync`, `add`, `audit`, `contracts`, `drift`, `guide` touch an LLM.

**Why:**
1. **Speed.** Deterministic commands run in ms–s, not minutes.
2. **Reproducibility.** Same input → same output, every time.
3. **Cost.** Users can run `status`, `validate`, `coverage` hundreds of times per day free.
4. **Debuggability.** When something's wrong, deterministic commands are where you diagnose.

**Consequence:** Every derived view must be expressible as a graph transformation over `manifest.json`. If it isn't, either (a) put the reasoning in the per-file analyzer prompt so it lands in frontmatter, or (b) add it as an explicit LLM pass with its own prompt.

---

## ADR-005: Frontmatter is the contract, not prose

**Decision:** Every per-file analysis starts with YAML frontmatter. The body is human-readable markdown; the frontmatter is machine-parseable. Downstream commands read frontmatter, never body text.

**Why:** Downstream reliability. Parsing prose is fragile and expensive; parsing YAML is trivial and deterministic. The frontmatter is what turns this from "documentation generator" into "queryable knowledge base."

**Consequence:** Any new structured field goes in frontmatter first, prose second. The prompt always requires the model to restate frontmatter claims in both places.

---

## ADR-006: Closed intent taxonomy

**Decision:** `schemas/intents.v1.yaml` is a fixed, closed vocabulary. The LLM picks from it or picks nothing — it never invents IDs. Unknown IDs fail validation.

**Why:** Open vocabularies become tag soup immediately. Every search becomes a fuzzy match. Aggregation breaks. With a closed vocabulary, "show me every file tagged `security-auth`" is a perfect query.

**Consequence:** Adding tags requires updating the YAML file and bumping the taxonomy version. This is intentional friction — it forces deliberation about whether a new tag is actually useful.

---

## ADR-007: All 7 compliance frameworks always on

**Decision:** HIPAA, SOC 2, ISO 27001, GDPR, CCPA/CPRA, CMIA, and California misc are all included in the base taxonomy and enabled by default. Projects opt *out* via `docs.config.json.compliance.frameworks`.

**Why:**
1. Frameworks overlap heavily — the same delete endpoint satisfies GDPR Art.17 and CCPA 1798.105 and ISO 27001 A.8.10 and SOC 2 P4.1. Cross-framework mapping is exactly what compliance teams want.
2. Opt-in-by-default means most users never see compliance output. Opt-out means every user gets it and can narrow later.
3. The cost of including a framework the project doesn't use is near-zero (the model just won't tag anything).

**Consequence:** `COMPLIANCE.md` for a non-regulated codebase will be mostly empty — that's fine, it's honest, and the "controls with no visible implementation" list is still informative.

---

## ADR-008: Compliance tags require evidence

**Decision:** Every `compliance-*` intent entry must include an `evidence` string quoting the code or citing a contract, AND a `controls` list with at least one specific framework control ID. Enforced by JSON Schema conditional validation.

**Why:** Compliance output is consumed by auditors and lawyers. False positives are expensive. Requiring evidence forces the model to be specific — if it can't quote the code, it shouldn't tag. Under-tagging is safe; over-tagging undermines audit trust.

**Consequence:** Every `COMPLIANCE.md` report carries a "NOT LEGAL ADVICE" disclaimer. Every tag is reviewable. The next AI should never relax this rule.

---

## ADR-009: "Under-tag" over "over-tag"

**Decision:** When the model is uncertain, the prompt tells it to drop the tag, not add it with low confidence.

**Why:** Compliance and audit workflows treat tags as claims. A wrong claim is worse than a missing one — missing claims show up in coverage gap reports and can be added later. Wrong claims give a false sense of security.

This principle applies beyond compliance: `audit` findings, `contracts` verdicts, `drift` verdicts — all prefer silence to false confidence.

---

## ADR-010: Structured `assumes` / `enforces` enables cross-file bug hunting

**Decision:** Section 5 of every analysis produces machine-readable `assumes` (what the file requires of upstream) and `enforces` (what the file guarantees to downstream) entries in frontmatter.

**Why:** This turns cross-file bug hunting from vague "look at everything" into a graph problem: walk `depends_on`, match `enforces` against downstream `assumes`, flag mismatches. It's the single highest-leverage structural change possible for AI-powered static analysis of large codebases.

**Consequence:** `contracts` and `drift` commands are only useful because this structure exists. Losing or weakening the `assumes`/`enforces` prompt rules kills the bug-hunting value.

---

## ADR-011: `audit` reads analyses only, never source

**Decision:** The adversarial per-group audit pass reads the per-file analyses (structured frontmatter + prose) rather than reading source files directly.

**Why:**
1. **Token economy.** Analyses are pre-digested and smaller than source.
2. **Forces reliance on contracts.** If the model has to re-read the source, it'll pattern-match on code style instead of reasoning about invariants. Reading analyses forces it to reason about `assumes` / `enforces` / `risks` / `top_suspicion`.
3. **Cache-friendly.** The analyses don't change between audit runs unless the source changes, so re-running is fast.

**Consequence:** Audit quality is bounded by analysis quality. Improving the per-file prompt has direct leverage on every audit finding.

---

## ADR-012: Cold-start needs scope filters

**Decision:** `init` supports `--groups`, `--kinds`, `--limit`, `--since <git-ref>`, `--estimate`, `--dry-run`.

**Why:** A real codebase is 500+ files. A single unfiltered `init` would cost tens of millions of tokens and minutes to hours of wall time. Users need to (a) try a small slice first to validate prompt output, (b) bootstrap one group at a time, (c) see a cost ballpark before spending. Without these flags, cold start is a cliff.

**Consequence:** Any future command that processes the whole tree should support the same filters via `src/scope.mjs`.

---

## ADR-013: Self-hosted documentation in `analyses/`

**Decision:** mhng-repo-mind documents itself with hand-written markdown files in `analyses/`, matching the shape mhng-repo-mind produces for any target. There's no generated per-file forensic analysis of mhng-repo-mind's own source (yet).

**Why:**
1. **Bootstrapping.** A future AI session needs a map of the repo. Hand-written is faster to produce and richer for small repos than generated forensic analyses.
2. **Eating our own dogfood.** The shape (`FOR_FUTURE_AI.md` → `ARCHITECTURE.md` → `SPEC.md` → `DECISIONS.md` → `ROADMAP.md` → `INDEX.md`) is exactly what a user's repo would produce after `init` + `guide`.
3. **Evolution.** Once the CLI works end-to-end, a future pass can run `mhng-repo-mind init` on this repo itself and replace these files with generated per-file analyses, keeping the architecture / decision / roadmap docs as guides.

**Consequence:** Keep this folder updated. When you add a feature, update `ROADMAP.md`. When you make a non-obvious choice, add it here. When you break the spec, update `SPEC.md` and `CHANGELOG.md`.

---

## ADR-014: Prompts live in plain markdown, not in code

**Decision:** Prompts are `prompts/*.md` files. They're loaded as strings by `llm.mjs` and substituted with placeholders where needed. They are never embedded in JavaScript source or agent definitions.

**Why:** Portability. Any user can port mhng-repo-mind to a different LLM runner by writing an adapter that reads the same `prompts/` files. Any user can read and edit the prompts without grepping code. Any reviewer can diff prompt changes in a PR without parsing through escape characters.

**Consequence:** If a prompt needs logic (conditional sections, loops), that logic goes in the slash command or the CLI — not in the prompt file itself. The prompt file is always static text.

---

## ADR-015: Manifest rebuild after every LLM pass

**Decision:** After `sync` or `add` completes, the CLI always calls `buildManifest()` deterministically, regardless of what the LLM wrote.

**Why:** The LLM can produce malformed frontmatter, miss a file, or fail silently. The manifest rebuild is the ground truth — it rehashes every source, re-reads every frontmatter, re-resolves every edge. This catches drift between "what the LLM claims to have done" and "what's actually on disk."

**Consequence:** Any new LLM-powered command that modifies analyses must call `buildManifest()` + `writeManifest()` at the end.

---

## ADR-016: No auto-fix, only surface

**Decision:** `audit`, `contracts`, `drift` all report findings. None of them auto-apply fixes.

**Why:** Fixes need human judgment. Auto-fix in a security / compliance / bug context is actively dangerous — it encourages rubber-stamping. mhng-repo-mind's value is making humans better at reviewing, not replacing them.

**Consequence:** If a future AI is asked to add auto-fix: push back. Suggest a separate command (`mhng-repo-mind fix-suggest`) that writes patches to a file for human review instead.
