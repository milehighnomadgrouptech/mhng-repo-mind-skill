# Changelog

All notable changes to this project will be documented here.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and the spec version follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

> **A note on versioning.** Specs v1.1 through v1.6 were internal development milestones on the path to the first public release. They were never cut as separate public releases. The first public release is **v1.7.0**, which includes everything from specs v1.0 through v1.7 as a single shipped artifact. The spec milestones are retained in the v1.7.0 release notes below as a historical guide to how the data model evolved.

## [Unreleased]

_Nothing yet._

## [1.7.0] - 2026-04-08 — first public release

This release bundles **every feature built across internal spec milestones v1.0 → v1.7** into a single public artifact. Features are grouped by the spec milestone that introduced them.

### Fixed — Windows compatibility
- **`src/llm.mjs`** — `runClaudeCommand` and `runClaudePrompt` now use `shell: true` and quote arguments via a small `shellQuote` helper so the `claude` CLI shim resolves on Windows (`.cmd`).
- **`src/commands/doctor.mjs`** — preflight `claude --version` check uses shell mode for the same reason.
- **`src/commands/validate.mjs`** — switched Ajv import from `ajv` to `ajv/dist/2020.js` (draft 2020-12 build) to match the rest of the codebase.

### Added — one-shot pipeline runner (`auto`)
- **`mhng-repo-mind auto`** — end-to-end pipeline runner for unattended runs on a fresh target. Bootstraps a default `docs.config.json` if the target has none, then walks: doctor → init → manifest → validate → symbols → glossary → entry-points → dead-code → graph → intents → compliance → coverage → sources → overview → contracts → audit → process --discover → process. Critical steps abort on failure; best-effort steps log warnings and continue. Flags: `--skip-audit`, `--skip-process`, `--output <dir>`.

### Added — process trees (spec milestone v1.7)
- **`mhng-repo-mind process`** — generates multi-file deep-dives for user-declared business processes. A process is a recursive tree of stages; the command produces one `OVERVIEW.md` per non-leaf node and one `<stage>.md` per leaf, all cross-linked with breadcrumbs and prev/next siblings. Bottom-up generation (leaves → branches → root) so parents always reference real children.
- **`mhng-repo-mind process --discover`** — read-only LLM pass that clusters files by folder/dependency/tag signal and writes `<output_dir>/processes/DISCOVERED.md` proposing processes for the human to review. Never edits `docs.config.json`. Never edits the manifest. AI proposes; human ratifies; names lock in.
- **`processes` config block** — recursive `processNode` schema in `docs.config.json`. Each top-level entry is an independent tree. Leaf stages match files via `file_patterns` and/or `tags_include` (intent IDs); branches inherit the union of their descendants. Optional `lock`, `compliance_relevant`, and top-level `group` for INDEX rendering.
- **`processes` manifest block** — recursive `processManifestNode` shape in `manifest.json`. Mirrors the declared tree with computed `coverage_sha` per node. Branches hash over their children; leaves hash over (sorted source `analysis_sha`s + prompt sha + node config).
- **Frontmatter additions** — optional `processes: [...]` array on per-file forensic analyses; new `process_node` block on stage .md files.
- **Resolver** (`src/processes.mjs`), prompts (`prompts/process.md`, `prompts/process-discover.md`), and slash command (`.claude/commands/docs-process.md`) — all pure-Node resolution, LLM only for prose.

### Added — full dbt borrowed feature set (spec milestone v1.6)
- **Indirect selection** — `--indirect-selection eager|cautious|buildable|none` flag on `select`. When a node matches, automatically pull in tests that import it. Borrowed from dbt.
- **Spec-state hashing** (`src/spec-state.mjs`) — computes a single 16-hex hash from every prompt, schema, taxonomy, and `docs.config.json`. `sync` and `status` report `spec-stale` files: source SHA matches but the prompt/schema that produced the analysis has changed. **The single most important invalidation fix** — makes prompt iteration safe.
- **`mhng-repo-mind contract-diff <ref>`** — compares current manifest's `enforces` blocks against a previous snapshot or git ref. Classifies changes as `safe` / `suspicious` / `breaking`. Breaking changes require a `version` bump or `deprecation_date`; otherwise exits non-zero. CI gate against silent contract regressions.
- **`@requires` decorator chain** (`src/requires.mjs`) — composable preflight chain.
- **Structured event system** (`src/events.mjs`) — typed events with stable codes. Console + JSONL reporters. Honors `REPO_MIND_LOG_LEVEL`.
- **Hooks lifecycle** (`src/hooks.mjs`) — `docs.config.json.hooks` accepts `on_<command>_<phase>` arrays of shell commands. `on_any_*` fires for every command. Start hooks fatal on failure, end hooks non-fatal warnings.
- **GraphQueue executor** (`src/graph-queue.mjs`) — topologically-ordered priority queue for parallel deterministic graph traversals.
- **Task hierarchy** (`src/tasks/base.mjs`) — `BaseTask → ConfiguredTask → ManifestTask → GraphRunnableTask → LLMTask` borrowed from dbt's `task/`. Layered ON TOP of the flat `src/commands/` pattern; both styles coexist.

### Added — access modifiers + versioning + deprecation (spec milestone v1.6)
- **Frontmatter `access` field** — `public` (default), `protected`, or `private`. Borrowed from dbt's access modifiers. Mirrored into the manifest.
- **Frontmatter `version`, `latest_version`, `deprecation_date`, `deprecation_reason` fields** — borrowed from dbt's model versioning. `latest_version` is computed by the manifest builder.
- **`mhng-repo-mind access-check`** — flags cross-group imports that violate `private` or `protected`. Writes `ACCESS-CHECK.md`. CI governance gate.
- **`mhng-repo-mind deprecated`** — lists nodes past their `deprecation_date` with consumer counts. Writes `DEPRECATED.md`. `--strict` fails CI if any past-deprecation node still has consumers.

### Added — the rest of dbt's manifest pattern (spec milestone v1.5)
- **`unique_id` per node** — stable content-addressable identifier `<group>.<dirname>.<basename>`. Survives reorganization within a group; enables rename detection.
- **Top-level `metadata` envelope** — spec_version, schema_url, repo_mind_version, **`invocation_id`** (UUID per build), **`git_ref`** (`git rev-parse HEAD` of target_repo), `git_branch`, target_repo, output_dir. Every manifest is anchorable to a specific commit.
- **Typed `checksum` block** — `{ name: 'sha256', value: '...' }` replaces the flat `source_sha` as the primary form.
- **`first_seen_at` and `last_analyzed_at`** per node — preserved across rebuilds.
- **`disabled` field** — node stays in the manifest but is excluded from `audit`/`compliance`/`coverage`/`drift`.
- **`meta` field** — free-form user-extensible key-value pairs. Queryable via `mhng-repo-mind query --meta key=value`.
- **`semantic_parents` / `semantic_children`** — semantic edges beyond import edges. Inverted automatically.
- **`config_parents`** — separate list for config-driven edges.
- **`freshness` for declared sources** + **`mhng-repo-mind freshness` command** — borrowed from `dbt source freshness`. `warn_after_days` / `error_after_days` on external sources.

### Added — dbt-style features (spec milestone v1.4)
- **Selectors** (`src/selectors.mjs`) — dbt-style `--select` grammar: `tag:`, `intent:`, `control:`, `kind:`, `group:`, `path:`, `source:`, `model:`, `lock:`, `state:`, `exposure:`; `+prefix` upstream, `suffix+` downstream, `+depth+` bounded; `,` union, `&` intersect. New `mhng-repo-mind select <expr>` preview.
- **Run results** (`src/runs.mjs`) — every LLM pass appends a JSON artifact to `runs/<timestamp>.json`. New `mhng-repo-mind runs` history.
- **Sources with versioning** — parses lockfiles from **10 ecosystems** (npm, pnpm, yarn, poetry, pipenv, pip, cargo, go modules, rubygems, packagist). Writes `SOURCES.md` + `sources.json`. `mhng-repo-mind sources-diff` compares against a stored snapshot.
- **Exposures** — `docs.config.json.exposures` declares downstream consumers OUTSIDE the repo (sibling apps, dashboards, mobile clients). `drift` reports cross-repo impact.
- **Snapshots** — point-in-time captures of every generated report. `snapshot-diff` produces auditor-grade reports of compliance gained/lost between two snapshots. The SOC 2 surveillance audit deliverable.
- **Test coverage** — for every node with non-empty `enforces`, lists which test files import it. Flags uncovered enforces as unverified claims.
- **Materializations** — per-kind analysis depth and template selection. Token cost drops 40–60% on mixed-kind repos.
- **Macro / partial system** — `{{> name }}` placeholders in prompts inline `prompts/_partials/<name>.md` contents.
- **Frontmatter `source` field** — marks vendored/generated sources for lighter analysis.

### Added — provenance + locks + Claude Code integration (spec milestone v1.3)
- **Frontmatter `generated_by` block** — required on every machine-generated analysis. Records `tool`, `tool_version`, `spec_version`, `model`, `runner`, `prompt_path`, `prompt_sha`, `settings`, `generated_at`.
- **Frontmatter `lock: true` field** — marks an analysis as hand-tuned. `sync` and `add` skip locked files.
- **Manifest top-level `provenance` aggregate** — `models`, `tool_versions`, `spec_versions`, `prompt_shas`, `locked_count`.
- **New commands**: `provenance` (`PROVENANCE.md`), `locked` (`LOCKED.md`), `migrate --from <model> [--to <model>]` (re-analyze every file generated by an old model).
- **`/mhng-repo-mind` top-level slash command** at `.claude/commands/mhng-repo-mind.md`.
- **`.claude/settings.example.json`** — pre-built allowlist that pre-approves the tools mhng-repo-mind needs.
- **`docs/claude-code-integration.md`** — comprehensive guide.

### Added — intents + compliance (spec milestone v1.2)
- **Fixed intent taxonomy** at `schemas/intents.v1.yaml`: ~30 general intents across 6 families, 7 compliance frameworks (HIPAA, SOC 2, ISO 27001, GDPR, CCPA/CPRA, CMIA, California misc), ~140 total tag IDs.
- **Frontmatter schema** extended with `intents` array; compliance tags require `evidence` + `controls`.
- **New commands**: `intents` (`INTENTS.md`), `compliance` (`COMPLIANCE.md` + per-framework files; `--csv`, `--framework`, `--gaps`), `coverage` (evaluates `required_controls` + `intent_rules`).
- **`query`** extended with `--intent <id>` and `--control <id>`.

### Added — derived views + bug-hunting (spec milestone v1.1)
- **Derived view commands** (no LLM, pure graph): `glossary`, `entry-points`, `dead-code`, `graph`.
- **Task-oriented guides**: `guide <question>` + `/docs-guide` + `prompts/guide.md`.
- **Structured contracts** in file-analysis frontmatter: `assumes`, `enforces`, `risks`, `top_suspicion`.
- **New prompts**: `audit-group.md`, `drift-check.md`, `contracts-check.md`.
- **New commands**: `audit`, `contracts`, `drift <ref>`, `lint`, `triage`, `props`.
- **Manifest v1.1**: nodes carry `assumes` / `enforces` / `risks` / `top_suspicion` mirrored from frontmatter.
- **Config**: `docs.config.json.tools` for declaring typecheck/lint/test commands.

### Added — base (spec milestone v1.0)
- Initial scaffold: README, license, schemas, prompts, slash commands, file-analyzer agent.
- Spec `v1`: frontmatter schema, manifest schema, docs.config schema.
- 8-section forensic file-analysis template.
- Group summary and INDEX templates.
- Slash commands: `/docs-init`, `/docs-sync`, `/docs-add`, `/docs-query`.
- CLI: `init`, `sync`, `add`, `query`, `status`, `validate`, `manifest`, `symbols`, `doctor`, `install-hooks`, `watch`.
- Cold-start flags: `--groups`, `--kinds`, `--limit`, `--since`, `--estimate`, `--dry-run`.

### Changed — config (v1.7.0)
- **`output_dir` is now optional** in `docs.config.json`; defaults to sibling directory `<basename(target_repo)>-mhng-repo-mind` next to the target repo. Removed from the schema's `required` array and from `docs.config.example.json`. Defaulting logic lives in `src/config.mjs`.

### Changed — license (v1.7.0)
- **License switched from MIT to Functional Source License v1.1 with MIT Future (FSL-1.1-MIT).** Copyright holder set to **Logan Banks — Mile High Nomad Group**. The change locks in the open-core business model: anyone can use, modify, and self-host mhng-repo-mind for any non-competing purpose, and each release automatically converts to the standard MIT License two years after publication.
- **`LICENSE`** replaced with the full FSL-1.1-MIT text.
- **`package.json`** `license` field updated to `SEE LICENSE IN LICENSE`. Description rewritten. Keywords expanded.
- **`README.md`** License section rewritten with the FSL TL;DR.
- **`CONTRIBUTING.md`** updated with a contribution-licensing clause.
</content>
</invoke>