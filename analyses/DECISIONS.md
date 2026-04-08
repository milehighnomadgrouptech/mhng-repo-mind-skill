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

## ADR-003: LLM delegation via `claude -p "/slash-command"`

**Decision:** LLM-powered CLI commands shell out to Claude Code with a slash-command argument. The slash command lives in `.claude/commands/docs-*.md` and contains the orchestration logic. The CLI is a thin shim.

**Why:**
1. Orchestration logic (fan-out, retries, parallel batches, subagent spawning) is complex and already solved by Claude Code.
2. The slash commands are also usable directly inside Claude Code, so `mhng-repo-mind` has two user experiences for free.
3. If the user wants to swap LLM runners, they set `REPO_MIND_LLM_CMD`. The prompts in `prompts/` are portable.

**Alternatives considered:**
- Call the Anthropic SDK directly from Node (would lose Claude Code's orchestration and re-invent subagent management)
- Use Python (adds a runtime dependency; Node matches the target audience better)

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
