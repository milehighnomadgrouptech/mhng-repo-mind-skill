# Changelog

All notable changes to this project will be documented here.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and the spec version follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

> **Release note (2026-04-08)**: Specs v1.1 through v1.7 were developed incrementally on the `initial-release` branch and cut as point releases on the same day. Dates below reflect the cut date, not the original development date of each feature.

## [Unreleased]

_Nothing yet._

## [1.7.0] - 2026-04-08

### Added — process trees
- **`mhng-repo-mind process`** — generates multi-file deep-dives for user-declared business processes. A process is a recursive tree of stages; the command produces one `OVERVIEW.md` per non-leaf node and one `<stage>.md` per leaf, all cross-linked with breadcrumbs and prev/next siblings. Bottom-up generation (leaves → branches → root) so parents always reference real children.
- **`mhng-repo-mind process --discover`** — read-only LLM pass that clusters files by folder/dependency/tag signal and writes `<output_dir>/processes/DISCOVERED.md` proposing processes for the human to review. Never edits `docs.config.json`. Never edits the manifest. AI proposes; human ratifies; names lock in.
- **`processes` config block** — recursive `processNode` schema in `docs.config.json`. Each top-level entry is an independent tree. Leaf stages match files via `file_patterns` and/or `tags_include` (intent IDs); branches inherit the union of their descendants. Optional `lock`, `compliance_relevant`, and top-level `group` for INDEX rendering.
- **`processes` manifest block** — recursive `processManifestNode` shape in `manifest.json`. Mirrors the declared tree with computed `coverage_sha` per node (leaves hash over sorted source `analysis_sha`s + prompt sha + node config; branches hash over their children). Powers idempotent regeneration.
- **Frontmatter additions** — optional `processes: [{process, stage, breadcrumb, kind, md_path}]` array on per-file forensic analyses, so each file links back to every process+stage it belongs to. New `process_node` block on stage .md files carries breadcrumb, parent/child links, prev/next siblings, source_files, and coverage_sha.
- **Resolver** (`src/processes.mjs`) — pure Node. Validates the config tree (unique sibling names, leaf coverage requirement), walks each tree, matches sources via picomatch-style globs against the manifest, builds a tag→files index from `intents`, computes hashes, and writes a work plan to `<output_dir>/.mhng-repo-mind-process-plan.json` for the slash command to consume.
- **Prompts** — `prompts/process.md` (leaf + branch + root templates with hard rules: never invent stages, never read source code directly, bottom-up generation, length caps) and `prompts/process-discover.md` (clustering rules, draft format, ≤12 proposals per run, ≤4 levels deep).
- **Slash command** — `.claude/commands/docs-process.md` orchestrates generation and discovery modes. Generation mode walks bottom-up, skips locked and unchanged nodes, and writes only under `<output_dir>/processes/<process_name>/`.
- **Manifest builder integration** — `src/manifest.mjs` now re-resolves the process trees from config on every build and reads on-disk stage frontmatter for the actual `coverage_sha` and `lock` state. Bad config doesn't fail the manifest build (surfaces via `validate` and `process` instead).
- **Schema additions** — `processNode` in `docs.config.schema.json`, `processManifestNode` in `manifest.schema.json`, `processes` and `process_node` in `frontmatter.schema.json`. All additive; no existing fields touched.

### Changed — config
- **`output_dir` is now optional** in `docs.config.json`; defaults to sibling directory `<basename(target_repo)>-mhng-repo-mind` next to the target repo. Removed from the schema's `required` array and from `docs.config.example.json`. Defaulting logic lives in `src/config.mjs`.

### Changed — license
- **License switched from MIT to Functional Source License v1.1 with MIT Future (FSL-1.1-MIT).** Copyright holder set to **Logan Banks — Mile High Nomad Group**. The change locks in the open-core business model: anyone can use, modify, and self-host mhng-repo-mind for any non-competing purpose, and each release automatically converts to the standard MIT License two years after publication.
- **`LICENSE`** replaced with the full FSL-1.1-MIT text.
- **`package.json`** `license` field updated to `SEE LICENSE IN LICENSE`. Description rewritten with the official 343-character pitch. Keywords expanded for discoverability (compliance, gdpr, hipaa, soc2, iso27001, ccpa, audit, dbt, fair-source).
- **`README.md`** License section rewritten with the FSL TL;DR and a commercial-licensing pointer.
- **`CONTRIBUTING.md`** updated with a contribution-licensing clause: contributions are submitted under FSL-1.1-MIT terms.

### Fixed — Windows compatibility
- **`src/llm.mjs`** — `runClaudeCommand` and `runClaudePrompt` now use `shell: true` and quote arguments via a small `shellQuote` helper so the `claude` CLI shim resolves on Windows (`.cmd`).
- **`src/commands/doctor.mjs`** — preflight `claude --version` check uses shell mode for the same reason.
- **`src/commands/validate.mjs`** — switched Ajv import from `ajv` to `ajv/dist/2020.js` (draft 2020-12 build) to match the rest of the codebase.

## [1.6.0] - 2026-04-08

### Added — full dbt borrowed feature set
- **Indirect selection** — `--indirect-selection eager|cautious|buildable|none` flag on `select`. When a node matches, automatically pull in tests that import it. Borrowed from dbt's indirect selection. `eager` (default) includes all tests of any selected node; `cautious` only includes a test if all of its deps are also selected; `none` disables.
- **Spec-state hashing** (`src/spec-state.mjs`) — computes a single 16-hex hash from every prompt, schema, and taxonomy file plus `docs.config.json`. `sync` and `status` now report `spec-stale` files: source SHA matches but the prompt/schema that produced the analysis has changed. **The single most important invalidation fix** — makes prompt iteration safe.
- **`mhng-repo-mind contract-diff <ref>`** — compares current manifest's `enforces` blocks against a previous snapshot or git ref. Classifies changes as `safe` / `suspicious` / `breaking`. Breaking changes require a `version` bump or `deprecation_date`; otherwise exits non-zero. CI gate against silent contract regressions.
- **`@requires` decorator chain** (`src/requires.mjs`) — composable preflight chain: `requireConfig`, `requireManifest`, `requireGitRepo`, `requireOutputWritable`. Available for new commands; existing commands can opt in incrementally.
- **Structured event system** (`src/events.mjs`) — typed events with stable codes (A=preflight, P=parse, M=manifest, L=LLM, V=validate, R=report, H=hook, X=error). Console reporter (pretty colored stderr) and JSONL reporter (structured logs to `runs/<id>.events.jsonl`). Honors `REPO_MIND_LOG_LEVEL`.
- **Hooks lifecycle** (`src/hooks.mjs`) — `docs.config.json.hooks` accepts `on_<command>_<phase>` arrays of shell commands. Schema-validated. `on_any_*` fires for every command. Start hooks fatal on failure, end hooks non-fatal warnings. Hooks run with env vars for config path, command name, target/output paths.
- **GraphQueue executor** (`src/graph-queue.mjs`) — topologically-ordered priority queue for parallel deterministic graph traversals. Used internally by the new task hierarchy.
- **Task hierarchy** (`src/tasks/base.mjs`) — `BaseTask → ConfiguredTask → ManifestTask → GraphRunnableTask → LLMTask` borrowed from dbt's `task/`. Each layer adds one capability and inherits the previous. **Layered ON TOP of the flat `src/commands/` pattern, not replacing it** — both styles coexist. New commands can opt in for the shared lifecycle. Documented in `src/tasks/README.md`.

### Added — access modifiers + versioning + deprecation
- **Frontmatter `access` field** — `public` (default), `protected`, or `private`. Borrowed from dbt's access modifiers. Mirrored into the manifest.
- **Frontmatter `version`, `latest_version`, `deprecation_date`, `deprecation_reason` fields** — borrowed from dbt's model versioning. `latest_version` is computed by the manifest builder by grouping nodes whose `unique_id` shares the same family (everything before `.v<n>`).
- **`mhng-repo-mind access-check`** — pure-Node command that walks the manifest and flags every cross-group import that violates a `private` or `protected` access modifier. Writes `ACCESS-CHECK.md`. Exits non-zero on violations. Designed as a CI governance gate.
- **`mhng-repo-mind deprecated`** — pure-Node command that lists every node past its `deprecation_date` (with consumer count) plus upcoming deprecations within `--warn-days` (default 30). Writes `DEPRECATED.md`. With `--strict`, exits non-zero if any past-deprecation node still has consumers.
- **Schema additions**: `access`, `version`, `latest_version`, `deprecation_date`, `deprecation_reason` on both `frontmatter.schema.json` and `manifest.schema.json`.

## [1.5.0] - 2026-04-08

### Added — the rest of dbt's manifest pattern
- **`unique_id` per node** — stable content-addressable identifier built from `<group>.<dirname>.<basename>`. Survives reorganization within a group. Enables rename detection (`unique_id` matches but `source_file` differs) and stable cross-snapshot citations in audit findings.
- **Top-level `metadata` envelope** — wraps spec_version, schema_url, repo_mind_version, **`invocation_id`** (UUID per build), **`git_ref`** (`git rev-parse HEAD` of target_repo at build time), `git_branch`, target_repo, output_dir. Every manifest is now anchorable to a specific commit. Auditors get a real chain of custody.
- **Typed `checksum` block** — `{ name: 'sha256', value: '...' }` replaces the flat `source_sha` as the primary form. `source_sha` still emitted for backwards compatibility. Lets future versions switch hash algorithms (BLAKE3, content-aware) without a manifest migration.
- **`first_seen_at` and `last_analyzed_at`** per node — `first_seen_at` set on creation, **preserved across rebuilds** by reading the previous manifest. Combined with `last_analyzed_at`, gives the age of every documented file. Powers "show me code that's been around 6+ months without re-audit" queries.
- **`disabled` field** — node stays in the manifest and is queryable but **excluded from `audit`, `compliance`, `coverage`, `drift`**. For archived/deprecated code that should remain documented but not actively reasoned about.
- **`meta` field** — free-form user-extensible key-value pairs on every node. mhng-repo-mind never reads it directly — it's the extension point. New `mhng-repo-mind query --meta key=value` filter.
- **`semantic_parents` / `semantic_children`** — semantic edges that exist BEYOND import edges. Inverted automatically by the manifest builder. Lets `audit` and `drift` walk relationships like "this file's `assumes` points at that file's `enforces`" or "this Next.js page is configured by `next.config.js`".
- **`config_parents`** — separate list for config-driven edges (page → next.config.js, model → schema.prisma, etc.).
- **`disabled_count` in provenance aggregate** — top-level counter for disabled nodes.
- **`freshness` for declared sources** + **`mhng-repo-mind freshness` command** — borrowed from `dbt source freshness`. Each declared source in `docs.config.json.sources` can carry `freshness: { warn_after_days, error_after_days }` and `last_reviewed: <date>`. The command writes `FRESHNESS.md` and exits non-zero on violations. Useful for external APIs and partner integrations that have no lockfile to track.
- **Frontmatter schema additions**: `disabled`, `meta`, `first_seen_at`, `semantic_parents`, `config_parents`.
- **Manifest schema additions**: top-level `metadata` envelope, all the new node fields.
- **`docs.config.json` sources gain** `last_reviewed` + `freshness` fields with their own JSON Schema validation.

## [1.4.0] - 2026-04-08

### Added — dbt-style features
- **Selectors** (`src/selectors.mjs`) — dbt-style `--select` expression grammar:
  - `tag:`, `intent:`, `control:`, `kind:`, `group:`, `path:`, `source:`, `model:`, `lock:`, `state:`, `exposure:`
  - `+prefix` upstream walks, `suffix+` downstream walks, `+depth+` bounded walks
  - Comma = union, `&` = intersect
  - New `mhng-repo-mind select <expr>` command for previewing selector results
- **Run results** (`src/runs.mjs` + `src/commands/runs.mjs`) — every LLM pass appends a JSON artifact to `runs/<timestamp>.json` with command, args, model, duration, files processed, status. New `mhng-repo-mind runs` command shows the history table.
- **Sources with versioning** (`src/sources-detect.mjs` + `src/commands/sources.mjs` + `src/commands/sources-diff.mjs`) — parses lockfiles from **10 ecosystems** (npm, pnpm, yarn, poetry, pipenv, pip, cargo, go modules, rubygems, packagist) and writes:
  - `SOURCES.md` — markdown report grouped by ecosystem
  - `sources.json` — machine-readable manifest of every dependency with name, version, ecosystem, license, direct/transitive flag, lockfile origin
  - `mhng-repo-mind sources-diff [prev]` — compares current lockfile state against a stored snapshot, reporting added/removed/upgraded/downgraded packages. `--fail-on-downgrade` for CI.
  - Three input streams: in-graph files marked `source: true`, lockfile-detected packages, declared external sources from `docs.config.json.sources`
- **Exposures** (`src/commands/exposures.mjs`) — declared in `docs.config.json.exposures`, mirrored into the manifest, written to `EXPOSURES.md`. Downstream consumers OUTSIDE the repo (sibling apps, dashboards, mobile clients) so `drift` reports cross-repo impact.
- **Snapshots** (`src/commands/snapshot.mjs` + `src/commands/snapshot-diff.mjs`) — point-in-time captures of every generated report into `snapshots/<name>-<date>/`. `snapshot-diff` produces an auditor-grade report of compliance gained/lost, files added/removed, model usage changes, lock changes between two snapshots. The SOC 2 surveillance audit deliverable.
- **Test coverage** (`src/commands/test-coverage.mjs`) — for every node with non-empty `enforces`, lists which test files import it. Flags uncovered enforces as "unverified claims" — bug magnets. Pure graph traversal, no LLM.
- **Materializations** (`src/materializations.mjs`) — per-kind analysis depth and template selection. Default materializations for `module`, `component`, `type`, `test`, `config`, `sql`, `dist`, `generated`. Each one controls template, sections, token estimate, source flag, audit inclusion. Token cost on a 500-file repo drops 40-60% via lighter materializations on lower-value kinds.
- **Macro / partial system** (`src/partials.mjs` + `prompts/_partials/`) — `{{> name }}` placeholders in prompts inline contents from `prompts/_partials/<name>.md`. Three partials shipped: `intent-rules`, `compliance-evidence`, `frontmatter-rules`. Lets shared rules live in one place instead of being copy-pasted across every prompt.
- **Frontmatter `source` field** — marks an analysis as a vendored/generated source. Mirrored into manifest. Sources get lighter analysis and are skipped by `audit`.
- **Manifest top-level `exposures` field** mirrored from config.
- **`docs.config.json` gains** `exposures`, `sources`, `materializations` blocks.

## [1.3.0] - 2026-04-08

### Added — provenance + locks + Claude Code integration
- **Frontmatter `generated_by` block** — required on every machine-generated analysis. Records `tool`, `tool_version`, `spec_version`, `model`, `runner`, `prompt_path`, `prompt_sha`, `settings`, `generated_at`. Lets you answer "which files came from which model" without scanning every analysis.
- **Frontmatter `lock: true` field** — marks an analysis as hand-tuned. `sync` and `add` skip locked files entirely; humans can hand-edit them safely.
- **Manifest top-level `provenance` aggregate** — `models`, `tool_versions`, `spec_versions`, `prompt_shas`, `locked_count`. Enables fast queries across the whole graph.
- **New CLI commands**:
  - `provenance` → writes `PROVENANCE.md` grouping files by model with first/last seen timestamps and listing locked files
  - `locked` → writes `LOCKED.md` documenting the always-locked categories AND listing every per-file locked analysis
  - `migrate --from <model> [--to <model>]` → finds every file generated by an old model, queues them for re-analysis via `/docs-add`, skips locked files
- **`/mhng-repo-mind` top-level slash command** at `.claude/commands/mhng-repo-mind.md` — one entry point, dispatches to every subcommand. Works from any Claude Code session.
- **`.claude/settings.example.json`** — pre-built allowlist that pre-approves the tools mhng-repo-mind needs (Bash for `mhng-repo-mind`/`claude`/`git`, Read/Write/Edit in `analyses/`, etc.) and explicitly denies destructive operations (`rm -rf`, `git push`, edits to the target repo). Drops permission prompts to near-zero without disabling the permission system.
- **`docs/claude-code-integration.md`** — comprehensive guide covering CLI install, `/mhng-repo-mind` slash command, skill bundle, permission allowlist, `--dangerously-skip-permissions` for CI, lock-aware `PreToolUse` hooks, and CI integration.
- **`CLAUDE.md` updated** with a "Files you must NEVER edit" table and a "Provenance tracking" section.
- **`stale.mjs` updated** — `computeStale()` now returns a `locked` array. `sync` skips locked files and reports the count.

## [1.2.0] - 2026-04-08

### Added — intents + compliance
- **Fixed intent taxonomy** at `schemas/intents.v1.yaml`:
  - ~30 general intents across 6 families (correctness, safety, reliability, performance, interface, plumbing)
  - 7 compliance frameworks, all on by default: **HIPAA, SOC 2, ISO 27001, GDPR, CCPA/CPRA, CMIA, California misc**
  - ~140 total tag IDs, versioned per framework
- **Frontmatter schema** extended with `intents` array; conditional validation requires `evidence` + `controls` on compliance tags
- **Manifest schema** mirrors `intents` so downstream commands don't re-parse YAML
- **`prompts/file-analysis.md`** rewritten with a new section 5e — Intents:
  - Pick-from-taxonomy-only rule
  - Evidence mandatory rule
  - Cross-framework tagging rule (do not deduplicate)
  - Compliance-stricter-than-general rule
  - Not-legal-advice disclaimer
- **New CLI commands**:
  - `intents` → `INTENTS.md` grouped by tag, with evidence
  - `compliance` → `COMPLIANCE.md` + per-framework `COMPLIANCE-<fw>.md`, with mapped controls and gap analysis; `--csv`, `--framework`, `--gaps` flags
  - `coverage` → evaluates `required_controls` + `intent_rules` rules, writes `COVERAGE.md`, exits non-zero on failure
- **`query`** extended with `--intent <id>` and `--control <id>` for exact-tag retrieval
- **`docs.config.json`** gains a `compliance` object: `frameworks`, `jurisdictions`, `required_controls`, `intent_rules`
- **`src/taxonomy.mjs`** — loads the YAML once, exposes `isKnownIntent`, `isKnownControl`, `groupByIntent`, `groupByControl`

## [1.1.0] - 2026-04-08

### Added — derived views + task guides
- **Derived view commands** (no LLM, pure graph):
  - `glossary` → `GLOSSARY.md` of domain terms from type-kind nodes
  - `entry-points` → `ENTRY_POINTS.md` (nodes with empty `depended_on_by`)
  - `dead-code` → `DEAD_CODE.md` (candidate orphan internal modules)
  - `graph` → `GRAPH.md` with per-group Mermaid diagrams
- **Task-oriented guides**:
  - `guide <question>` CLI command
  - `/docs-guide` slash command
  - `prompts/guide.md` — narrative generation prompt reading analyses (not source)

### Added — bug-hunting
- **Structured contracts** in file-analysis frontmatter: `assumes`, `enforces`, `risks`, `top_suspicion`. Turns prose into a graph problem.
- **New sections** in `prompts/file-analysis.md`:
  - 5b — Adversarial Read (specific failure modes, not vague "invalid input")
  - 5c — Concurrency, Order, and State
  - 9 — Top Suspicion (forcing function)
- **New prompts**:
  - `prompts/audit-group.md` — per-group bug sweep, analyses only
  - `prompts/drift-check.md` — upstream/downstream contract check, three verdicts only
  - `prompts/contracts-check.md` — tiny oracle prompt for `(enforces, assumes)` pairs
- **New slash commands**: `/docs-audit`, `/docs-contracts`, `/docs-drift`.
- **New CLI commands**:
  - `audit` — cross-file adversarial sweep
  - `contracts` — match assume/enforce pairs across files
  - `drift <ref>` — PR-time downstream contract check
  - `lint` — run user's own tools, write `LINT.md` baseline
  - `triage` — human-curated `AUDIT.triage.md`
  - `props` — extract property-test candidates from `enforces`
- **Manifest v1.1**: nodes now carry `assumes` / `enforces` / `risks` / `top_suspicion` mirrored from frontmatter.
- **Config**: `docs.config.json.tools` for declaring typecheck/lint/test commands.

## [1.0.0] - 2026-04-08

### Added — base
- Initial scaffold: README, license, schemas, prompts, slash commands, file-analyzer agent.
- Spec `v1`: frontmatter schema, manifest schema, docs.config schema.
- 8-section forensic file-analysis template.
- Group summary and INDEX templates.
- `/docs-init`, `/docs-sync`, `/docs-add`, `/docs-query` slash commands.
- CLI: `init`, `sync`, `add`, `query`, `status`, `validate`, `manifest`, `symbols`, `doctor`, `install-hooks`, `watch`.
- Cold-start flags: `--groups`, `--kinds`, `--limit`, `--since`, `--estimate`, `--dry-run`.
</content>
</invoke>