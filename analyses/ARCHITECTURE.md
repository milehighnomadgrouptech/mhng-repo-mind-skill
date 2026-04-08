# Architecture

A technical map of mhng-repo-mind — every directory, every file type, how they connect, and what invariants hold across them.

## The three planes

mhng-repo-mind is organized around **three planes** that never cross:

1. **Deterministic plane** (`src/`, `schemas/`) — Node code, JSON Schemas, hash functions, graph builders. Never calls an LLM. Must be fast, reproducible, and correct.
2. **Prompt plane** (`prompts/`) — plain markdown files. Every prompt is a complete, self-contained instruction to an LLM subagent. No code. No templating beyond simple placeholders.
3. **Orchestration plane** (`.claude/commands/*.md`) — markdown prompt-template files. As of spec v1.8 these are **not** runtime slash commands; they are prompt source that the Node runner reads, renders `{{VARS}}` into, and pipes to `claude -p` over stdin. They retain their `.claude/commands/` location so Claude Code users can still browse them, but the runner does not invoke them as `/docs-init`.

The CLI (`bin/mhng-repo-mind` + `src/cli.mjs`) is the user's entry point to all three. It lives in the deterministic plane and, for LLM passes, inlines a prompt body and pipes it to `claude -p` via stdin.

## Top-level layout

```
mhng-repo-mind/
├── bin/
│   └── mhng-repo-mind                 # shebang entry → src/cli.mjs
├── src/
│   ├── cli.mjs                   # arg parser + dispatch
│   ├── config.mjs                # loads + validates docs.config.json
│   ├── walk.mjs                  # include/exclude globbing, kind/group resolution
│   ├── hash.mjs                  # sha256 helpers
│   ├── frontmatter.mjs           # parse YAML frontmatter from .md files
│   ├── manifest.mjs              # build/read/write manifest.json; resolve deps
│   ├── stale.mjs                 # diff sources vs manifest; downstream closure
│   ├── scope.mjs                 # --groups/--kinds/--limit/--since filters + cost estimator
│   ├── taxonomy.mjs              # load intents.v1.yaml; group manifest by intent/control
│   ├── llm.mjs                   # reads .claude/commands/docs-*.md as prompt source, renders {{VARS}} in Node, pipes inlined body to `claude -p` over stdin (override via REPO_MIND_LLM_CMD)
│   └── commands/
│       ├── init / sync / add      # LLM — base pipeline
│       ├── query                  # deterministic retrieval + --intent/--control
│       ├── status                 # deterministic staleness report
│       ├── validate               # JSON Schema validator
│       ├── manifest               # deterministic rebuild from disk
│       ├── symbols                # SYMBOLS.md reverse index
│       ├── doctor                 # preflight checks
│       ├── install-hooks          # git hook installer
│       ├── watch                  # fs.watch → sync
│       ├── audit                  # LLM — per-group adversarial sweep
│       ├── contracts              # LLM-filtered — assume/enforce pair check
│       ├── drift                  # LLM — PR-time downstream contract check
│       ├── lint                   # runs user's own tools
│       ├── triage                 # AUDIT.triage.md manager
│       ├── props                  # property-test seed extractor
│       ├── glossary               # GLOSSARY.md from type-kind nodes
│       ├── entry-points           # ENTRY_POINTS.md from empty depended_on_by
│       ├── dead-code              # DEAD_CODE.md candidates
│       ├── graph                  # GRAPH.md with Mermaid diagrams
│       ├── guide                  # LLM — task-oriented narrative
│       ├── intents                # INTENTS.md — why each file exists
│       ├── compliance             # COMPLIANCE.md + per-framework files
│       └── coverage               # required_controls + intent_rules gate
├── prompts/
│   ├── file-analysis.md          # THE master forensic prompt (10 sections)
│   ├── group-summary.md          # _SUMMARY.md synthesis
│   ├── index-builder.md          # INDEX.md router
│   ├── audit-group.md            # per-group bug sweep
│   ├── drift-check.md            # upstream/downstream verdict (safe|possibly_broken|broken)
│   ├── contracts-check.md        # (enforces, assumes) oracle (satisfied|not_satisfied|unknown)
│   └── guide.md                  # task guide narrative
├── schemas/
│   ├── frontmatter.schema.json   # per-file YAML frontmatter contract
│   ├── manifest.schema.json      # the dependency graph contract
│   ├── docs.config.schema.json   # user's config file contract
│   └── intents.v1.yaml           # the CLOSED tag taxonomy (7 compliance frameworks)
├── .claude/
│   ├── commands/                 # docs-init, -sync, -add, -query, -audit, -contracts, -drift, -guide
│   └── agents/
│       └── file-analyzer.md      # the subagent definition used for per-file work
├── .github/
│   └── workflows/
│       ├── validate.yml          # CI: validate the schemas parse
│       └── mhng-repo-mind.yml.example # copy into target repos for auto-sync
├── docs/
│   └── cli.md                    # full CLI reference
├── examples/
│   └── README.md                 # points at the mhng/packages reference run
├── analyses/                     # self-hosted documentation (this folder)
│   ├── FOR_FUTURE_AI.md
│   ├── ARCHITECTURE.md           # ← you are here
│   ├── SPEC.md
│   ├── DECISIONS.md
│   ├── ROADMAP.md
│   └── INDEX.md
├── docs.config.example.json      # copy → docs.config.json and edit
├── package.json                  # bin entry + minimal deps
├── README.md
├── CHANGELOG.md                  # KEEP UPDATED on every feature
├── LICENSE                       # MIT
├── CONTRIBUTING.md
└── CODE_OF_CONDUCT.md
```

## Data flow (in prose)

### Bootstrap: `init`
1. `cli.mjs` dispatches to `commands/init.mjs`.
2. `config.mjs` loads `docs.config.json` and validates it against `schemas/docs.config.schema.json`.
3. `walk.mjs` globs the target repo, resolving each file's `kind` and `group`.
4. `scope.mjs` applies `--groups`/`--kinds`/`--limit`/`--since`/`--estimate` filters.
5. If LLM work is needed, CLI writes a `.mhng-repo-mind-init-scope.json` plan file. The runner reads `.claude/commands/docs-init.md`, renders `{{VARS}}` (including the plan path), and pipes the inlined prompt body to `claude -p` over stdin. **No slash command is invoked**: Claude Code 2.1.92 does not execute slash commands when called via `-p`, and the Windows argv path mangles CRLF inside long bodies. Stdin sidesteps both.
6. The init prompt body instructs the model to read the plan and write one per-file `.md` per source file using `prompts/file-analysis.md` rules.
7. The init prompt body then runs `prompts/group-summary.md` per group and `prompts/index-builder.md` once for the whole repo.
   After the runner returns, the CLI verifies artifacts were actually written: zero artifacts or any "Unknown skill / Unknown command" leakage in stdout fails the command loudly.
8. Back in the CLI, `manifest.mjs` walks the resulting analyses, computes SHAs, parses frontmatter (including `intents`), resolves `depends_on` edges, inverts to `depended_on_by`, and writes `manifest.json`.

### Incremental: `sync`
1. `stale.mjs` computes added/modified/deleted sets by diffing current source SHAs against `manifest.json`.
2. It walks `depended_on_by` to find downstream files that may need refreshing.
3. The runner reads `.claude/commands/docs-sync.md`, inlines vars, and pipes the body to `claude -p` over stdin. The model amends only the affected analyses (preserving unchanged sections) and rewrites affected summaries.
4. `manifest.mjs` rebuilds deterministically — this catches any drift between what the LLM wrote and what the graph should look like.

### Bug hunting: `audit`, `contracts`, `drift`
- `audit`: reads manifest, groups nodes by `rolls_up_to`, inlines `.claude/commands/docs-audit.md` per group and pipes it to `claude -p` over stdin. The prompt reads analyses only — never source. Output: `<group>/_AUDIT.md` + aggregated `AUDIT.md`.
- `contracts`: builds `(upstream.enforces, downstream.assumes)` pairs from the manifest with a token-overlap heuristic, writes a plan file, inlines `.claude/commands/docs-contracts.md` and pipes it to `claude -p` over stdin (3-line oracle per pair). Output: `CONTRACTS.md` with satisfied/not_satisfied/unknown buckets.
- `drift`: takes a git ref, uses `git diff` to find changed files, walks `depended_on_by` to find downstream, inlines `.claude/commands/docs-drift.md` and pipes it to `claude -p` over stdin (safe/possibly_broken/broken verdicts). Output: `DRIFT.md`. Designed for CI on every PR.

### Intents & compliance
- `intents`: reads `manifest.nodes[*].intents`, groups by tag ID, writes `INTENTS.md`. Pure Node.
- `compliance`: reads `manifest.nodes[*].intents`, filters to `compliance-*` tags, groups by control ID per framework, writes `COMPLIANCE.md` + per-framework files. Pure Node.
- `coverage`: evaluates `docs.config.json.compliance.required_controls` and `compliance.intent_rules` against the manifest, exits non-zero on failure. Pure Node.

## Where the contracts live

Three schemas hold everything together:

### `schemas/frontmatter.schema.json`
Every per-file analysis starts with YAML frontmatter matching this schema. It carries:
- identity: `spec_version`, `source_file`, `analysis_path`, `group`, `kind`
- integrity: `source_sha`, `source_mtime`, `analyzed_at`
- graph: `exports`, `imports_internal`, `imports_external`, `related`, `rolls_up_to`
- bug-hunting: `assumes`, `enforces`, `risks`, `top_suspicion`
- intents: `intents[]` (with conditional validation — compliance tags require `evidence` + `controls`)

### `schemas/manifest.schema.json`
`manifest.json` is a dbt-style graph:
- `nodes[source_file]` carries all the per-file data copied from frontmatter + `depends_on`/`depended_on_by`/`analysis_sha`
- `summaries[path]` lists which files each `_SUMMARY.md` covers

### `schemas/intents.v1.yaml`
The closed taxonomy. Two layers:
- `general` — 6 families (correctness, safety, reliability, performance, interface, plumbing) with ~30 tags
- `compliance` — 7 frameworks (hipaa, soc2, iso27001, gdpr, ccpa, cmia, ca-misc) with ~140 control IDs, each with `source_version`

## Where the LLM enters and exits

The LLM only runs inside these commands:
- `init`, `sync`, `add` → per-file forensic analyses + group summaries + index
- `audit` → per-group bug sweep (reads analyses, not source)
- `contracts` → per-pair oracle on filtered `(enforces, assumes)` pairs
- `drift` → per-downstream verdict against a ref
- `guide` → narrative cross-file walkthrough

Every other command is deterministic Node. This is enforced by convention, not code — be ruthless about preserving it.

## Where state lives

- **On disk only.** No database, no in-memory cache between commands.
- `manifest.json` is the primary state.
- `AUDIT.triage.md` is human-curated state that survives audits.
- `docs.config.json` is user state (hand-edited, never written by mhng-repo-mind).
- `.mhng-repo-mind-*-plan.json` files are transient LLM handoff files, deleted after the LLM pass completes.

## Invocation invariants (spec v1.8)

- The runner reads `.claude/commands/docs-*.md` as **prompt source** and pipes the rendered body to `claude -p` over stdin. It does not call `claude -p "/docs-init"`. Reintroducing slash invocation will break Windows (CRLF mangling in argv) and silently no-op on Claude Code 2.1.92 (slash commands aren't executed via `-p`).
- Every LLM-delegating command verifies its artifacts post-run and fails loudly on `Unknown skill` / `Unknown command` leakage or zero artifacts.
- `doctor` round-trips a sentinel through the real runner to prove the pipeline works end-to-end. `--skip-llm-probe` is the offline escape hatch.
- `auto` gates each pipeline step on a per-step metric. `validate` is critical; init failure aborts the pipeline.
- A diagnostic + catch-all default group is always present so an unfamiliar tree never silently produces zero analyses.

## Branch-parity model (spec v1.8)

`docs.config.json.branches` selects how the docs store maps to target branches:
- `mode: "single"` (default) — one docs branch covers the target. Existing behavior, unchanged.
- `mode: "parity"` — the runner derives `target_repo` as `<target_worktree_root>/<docs-branch-name>` and enforces docs-branch-name === target-branch-name on every command. The new `branch-status`, `branch-sync`, `ask --branch`, and `watch --branch-mode` commands operate against this layout.

Every command takes a global `--ref <sha>` flag. If target HEAD doesn't match `manifest.metadata.git_ref` and `--ref` doesn't reconcile them, the command refuses to run.

`resolve-merge` is a git merge driver for analysis `.md` files; it reads the frontmatter `source_sha` from each side. Install lines for `.gitattributes` and `.git/config` are documented but not auto-installed.

`doctor` warns when `target_repo` lives inside the same git repo as the docs store and suggests a `git worktree add` layout to fix it.

## Self-hosted documentation

mhng-repo-mind documents itself in `analyses/` (this folder). It's the same shape mhng-repo-mind produces for any target — `INDEX.md` router, per-topic `.md` files, no generated per-file forensic analyses (the source is small enough that a hand-written architecture doc is more useful for bootstrapping a future AI session). See [FOR_FUTURE_AI.md](FOR_FUTURE_AI.md) for the entry point.
