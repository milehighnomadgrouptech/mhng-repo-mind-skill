---
name: mhng-repo-mind-skill
description: Use this skill to generate, maintain, query, or extend mhng-repo-mind documentation for any codebase. mhng-repo-mind produces forensic per-file analyses with structured YAML frontmatter, a dbt-style dependency graph (manifest.json), cross-file bug audits via assume/enforce contracts, compliance reports across HIPAA / SOC 2 / ISO 27001 / GDPR / CCPA / CMIA / California-misc with cited control IDs, task-oriented narrative guides, glossaries, entry-point and dead-code reports, dependency diagrams, contract drift detection, sources with versioning from lockfiles, exposures (downstream consumers outside the repo), snapshots, provenance tracking, and lifecycle hooks. Use whenever the user asks you to document a codebase, find cross-file bugs, generate compliance evidence, audit for security or privacy gaps, produce a project overview, trace how a feature works across files, check supply-chain dependencies, validate documentation freshness, or extend mhng-repo-mind itself.
---

# mhng-repo-mind skill — operating manual

You are operating the **mhng-repo-mind** documentation and reasoning harness on behalf of a user. This file is your complete operating manual. Follow it carefully — mhng-repo-mind has a strict spec, and producing output that doesn't conform breaks downstream commands.

## 0. The first thing you do, every time

When this skill is loaded, before you do anything else:

1. **Check whether the user has a `manifest.json` already.** Look for `<output_dir>/manifest.json` or `analyses/manifest.json` near the working directory. If it exists, READ IT FIRST. It's the single source of truth for the documentation graph and tells you everything that's already been analyzed, what shape the project takes, and what the user's configuration looks like.
2. **Check whether the `mhng-repo-mind` CLI is installed.** Try `mhng-repo-mind --version` via the Bash tool. If it succeeds, prefer the CLI for everything. If it fails, fall back to operating the prompts directly (Mode 2 below).
3. **Check whether `LOCKED.md` exists.** If it does, read it. It lists files you must never edit. Do not violate this even if explicitly asked — if asked to edit a locked file, stop and tell the user it's locked and how to unlock it.
4. **Check whether `PROVENANCE.md` exists.** If it does, skim it to see which model versions produced the existing analyses. If the user is on a newer Claude version, there may be analyses to migrate.
5. **Read `OVERVIEW.md` and `INDEX.md` if they exist.** These give you the project type and the topic router. Reading them first means you can answer most questions without loading per-file analyses at all.

This 30-second preflight saves you from repeating work, overwriting locked files, or producing inconsistent output.

## 1. What mhng-repo-mind is — your mental model

mhng-repo-mind is a **layered pipeline** that turns any codebase into structured, queryable, AI-maintainable documentation. The layers, from physical to logical:

```
target source repo
     │
     │   walk + glob (deterministic)
     ▼
list of source files with kind + group
     │
     │   one LLM subagent per file, prompt = file-analysis.md
     ▼
per-file .md analyses with strict YAML frontmatter
  ─ identity, imports/exports, line-by-line walkthrough
  ─ contracts (assumes / enforces), adversarial read
  ─ intents (closed taxonomy), top suspicion
  ─ provenance (which model, when, with what prompt SHA)
     │
     │   manifest.mjs deterministic build
     ▼
manifest.json — the dbt-style dependency graph
  ─ nodes[source].depends_on / depended_on_by
  ─ summaries[group] rolling up files
  ─ provenance + locks + access modifiers + versions
     │
     │   derived views (no LLM) and bug-hunting passes (LLM)
     ▼
OVERVIEW.md, INDEX.md, INTENTS.md,
COMPLIANCE.md + per-framework files,
AUDIT.md, CONTRACTS.md, DRIFT.md, COVERAGE.md,
GLOSSARY.md, ENTRY_POINTS.md, DEAD_CODE.md,
SYMBOLS.md, GRAPH.md, SOURCES.md, EXPOSURES.md,
PROVENANCE.md, LOCKED.md, FRESHNESS.md,
DEPRECATED.md, ACCESS-CHECK.md, CONTRACT-DIFF.md,
TEST_COVERAGE.md, GUIDES/, snapshots/, runs/
```

The split is the key idea: **deterministic Node code handles graphs, hashing, and validation; LLM agents handle prose; a fixed YAML/JSON schema connects them.** This is what makes the system incremental, queryable, and auditable.

The **manifest.json** is the single source of truth. When in doubt about anything in the codebase, query the manifest. It tells you which files exist, what they export, what they depend on, what contracts they declare, what intents they carry, what compliance controls they cite, what model produced their analysis, and when. You should rarely need to read every per-file analysis — the manifest is the index.

## 2. Three modes of operation

You will operate in one of three modes depending on what's available on the user's machine.

### Mode 1: Full CLI installed (preferred)
The `mhng-repo-mind` CLI is installed and on `$PATH`. You delegate work to it via the Bash tool. This is the canonical mode — the CLI knows how to spawn subagents in parallel, write the manifest correctly, capture run-results, and respect every spec invariant.

When in Mode 1, your job is to:
1. Decide which `mhng-repo-mind` subcommand the user needs
2. Build the right command line (with the right `--select` expression, `--limit`, `--groups`, etc.)
3. Run it via Bash
4. Read the output files it produces
5. Synthesize a human-friendly answer

You almost never need to read source code or per-file analyses directly in Mode 1 — the CLI's reports are what you summarize.

### Mode 2: CLI not installed
The CLI isn't available but the user wants mhng-repo-mind output anyway. You operate the prompts directly:
1. Walk the target tree yourself (Bash, Glob)
2. For each source file, follow `prompts/file-analysis.md` exactly to produce one `.md` with the correct frontmatter
3. After all per-file analyses are written, build the manifest yourself in JSON form by reading every frontmatter and computing `depends_on` from `imports_internal` (resolved against the source tree)
4. For derived views like `INTENTS.md`, `COMPLIANCE.md`, `GLOSSARY.md` — these are pure transformations of the manifest, you can produce them deterministically without further LLM calls

Mode 2 is slower and more error-prone but works on any machine that has Claude Code. Tell the user up front that you're in this mode and that running the CLI would be faster.

### Mode 3: Hybrid (most common)
The user has the CLI installed and has already run `mhng-repo-mind init` against their target. You're being asked a question. You read `INDEX.md`, `OVERVIEW.md`, the manifest, and one or two per-file analyses to answer it. You don't run new commands unless the user asks.

This is the **most common mode** and where you spend most of your time. Get good at navigating the documentation store without re-running anything.

## 3. The data model — what you're reading

### 3.1 Per-file analysis frontmatter (the contract)

Every per-file `.md` starts with YAML frontmatter conforming to `schemas/frontmatter.schema.json`. The full required + optional shape:

```yaml
---
# REQUIRED — identity
spec_version: "1"
source_file: packages/auth/src/session.ts
analysis_path: packages/auth/src/session.ts.md
group: auth
kind: module                # module | component | type | test | config | sql | dist | source | generated
exports: [getSession, requireSession, SessionError]
imports_internal: ["./api", "./schemas", "./types"]
imports_external: ["@workos-inc/authkit-nextjs", "zod"]
related: [packages/auth/src/middleware.ts]
rolls_up_to: packages/auth/_SUMMARY.md
source_sha: a1b2c3d4e5f60718  # 16 hex chars of sha256
source_mtime: 2026-04-07T18:32:14Z
analyzed_at: 2026-04-07T18:35:02Z

# REQUIRED — provenance (spec v1.3+)
generated_by:
  tool: mhng-repo-mind
  tool_version: "0.1.0-alpha.0"
  spec_version: "1"
  model: claude-opus-4-6
  model_alias: "Claude Opus 4.6"
  runner: claude-code
  prompt_path: prompts/file-analysis.md
  prompt_sha: 7f3a8b2c9d1e4f60
  spec_hash: 4e8b22fa5c11d39a   # invalidator that combines all prompts/schemas/taxonomy
  settings:
    temperature: null
    max_tokens: null
  generated_at: 2026-04-07T18:35:02Z
  duration_ms: 12340
  input_tokens: 8200
  output_tokens: 3500

# OPTIONAL — bug-hunting fields (spec v1.1)
assumes:
  - from: "session.getUser()"
    invariant: "returns non-null on a valid token"
    enforced_by: "not visible"
enforces:
  - for: "createSession() return"
    invariant: "id is a non-null UUID v4"
risks:
  - kind: unbounded-loop
    detail: "while(queue.length) without termination guard"
    severity: high
    location: "L42-L58"
top_suspicion: "createSession does not validate that the provided email matches the verified WorkOS profile"

# OPTIONAL — intents + compliance (spec v1.2)
intents:
  - id: security-auth
    confidence: high
    scope: file
    evidence: "Verifies WorkOS session token before route access"
  - id: compliance-gdpr
    confidence: high
    scope: file
    controls: ["art-32"]
    evidence: "Encrypts session tokens at rest via WorkOS"
  - id: compliance-soc2
    confidence: high
    scope: file
    controls: ["CC6.1", "CC6.7"]
    evidence: "Logical access via authentication + TLS in transit"

# OPTIONAL — locks + sources (spec v1.4)
lock: false
source: false

# OPTIONAL — graph extensions (spec v1.5)
first_seen_at: 2026-04-07T18:35:02Z
semantic_parents: [packages/types/src/user.ts]
config_parents: [next.config.js]
disabled: false
meta:
  owner: auth-team
  pii_severity: high
  last_security_review: "2026-Q1"

# OPTIONAL — access + versioning (spec v1.6)
access: public              # public (default) | protected | private
version: 2
latest_version: 2           # auto-computed by manifest builder
deprecation_date: null
deprecation_reason: null
---
```

**Critical rules for the frontmatter:**

1. **Required fields are required forever.** Bumping `spec_version` is the only legal way to change them.
2. **Compliance tags require `evidence` AND `controls`.** Conditional JSON Schema validation enforces this. If you cannot quote the specific line of code that implements a control, **do not tag it**.
3. **Intent IDs must come from `schemas/intents.v1.yaml`.** Never invent tag IDs. The taxonomy is closed.
4. **`assumes` and `enforces` must be concrete.** "id is a non-null UUID v4" ✅. "input is valid" ❌.
5. **Tag every framework that applies.** Do not deduplicate. A delete endpoint may carry GDPR Art.17 + CCPA 1798.105 + ISO 27001 A.8.10 + SOC 2 P4.1 simultaneously. This is correct and what auditors want.
6. **Under-tagging is safe; over-tagging undermines audit trust.** When uncertain, drop the tag.

### 3.2 The manifest

`manifest.json` mirrors every frontmatter field into a queryable graph plus computes the inverse edges and aggregates. Top-level shape:

```json
{
  "metadata": {
    "spec_version": "1",
    "schema_url": "https://mhng-repo-mind.dev/schemas/v1/manifest.schema.json",
    "repo_mind_version": "0.1.0-alpha.0",
    "invocation_id": "uuid",
    "generated_at": "2026-04-07T...",
    "git_ref": "abc123...",
    "git_branch": "main",
    "target_repo": "../my-app",
    "output_dir": "./analyses"
  },
  "spec_version": "1",
  "node_count": 85,
  "summary_count": 5,
  "nodes": {
    "packages/auth/src/session.ts": {
      "unique_id": "auth.packages.auth.src.session",
      "analysis": "packages/auth/src/session.ts.md",
      "group": "auth",
      "kind": "module",
      "checksum": { "name": "sha256", "value": "a1b2c3d4e5f60718" },
      "source_sha": "a1b2c3d4e5f60718",
      "first_seen_at": "2026-04-07T...",
      "last_analyzed_at": "2026-04-07T...",
      "exports": ["getSession", "requireSession"],
      "depends_on": ["packages/auth/src/api.ts"],
      "depends_on_raw": ["./api"],
      "depended_on_by": ["packages/auth/src/middleware.ts"],
      "rolls_up_to": "packages/auth/_SUMMARY.md",
      "assumes": [...],
      "enforces": [...],
      "risks": [...],
      "top_suspicion": "...",
      "intents": [...],
      "lock": false,
      "source": false,
      "disabled": false,
      "meta": {},
      "access": "public",
      "version": null,
      "latest_version": null,
      "deprecation_date": null,
      "semantic_parents": [],
      "semantic_children": [],
      "config_parents": [],
      "generated_by": { ... }
    }
  },
  "summaries": {
    "packages/auth/_SUMMARY.md": {
      "group": "auth",
      "covers": ["packages/auth/src/session.ts", "..."],
      "summary_sha": "..."
    }
  },
  "exposures": [
    { "name": "iOS app", "type": "client", "owner": "mobile", "depends_on": [...] }
  ],
  "provenance": {
    "models": {
      "claude-opus-4-6": { "count": 67, "first_seen": "...", "last_seen": "..." }
    },
    "tool_versions": ["0.1.0-alpha.0"],
    "spec_versions": ["1"],
    "prompt_shas": { "prompts/file-analysis.md": "7f3a8b2c9d1e4f60" },
    "locked_count": 3,
    "disabled_count": 0
  }
}
```

When asked any question about the project, **start by querying the manifest, not by reading files**. The manifest tells you 90% of what you need without spending a single LLM token on per-file analyses.

### 3.3 The intent taxonomy

`schemas/intents.v1.yaml` is a **closed** vocabulary with two layers:

**General intents** — six families:
- `correctness`: core-logic, data-transformation, state-management, validation, error-handling, edge-case-handling
- `safety`: security-auth, security-input, security-secrets, security-audit, privacy, pii
- `reliability`: concurrency, resilience, idempotency, observability, recovery, resource-management
- `performance`: performance-optimization, caching, lazy-loading, batching
- `interface`: public-api, ux-presentation, ux-interaction, i18n, accessibility
- `plumbing`: infrastructure, integration, test-harness

**Compliance intents** — seven frameworks, ~140 control IDs total:
- `compliance-hipaa` — 17 controls from 45 CFR §164 (Technical Safeguards + adjacent Administrative)
- `compliance-soc2` — 23 controls from AICPA TSC 2017 (CC + A + PI + C + P)
- `compliance-iso27001` — 41 controls from ISO 27001:2022 Annex A (Technological + select Organizational)
- `compliance-gdpr` — 33 controls from Regulation (EU) 2016/679 (Articles with code surface)
- `compliance-ccpa` — 19 controls from Cal. Civ. Code §1798.100+ + adjacent California statutes
- `compliance-cmia` — 6 controls from California Confidentiality of Medical Information Act
- `compliance-ca-misc` — 5 controls from SB-327 (IoT), SB-568 (minors), AB-375 (auto-renewal)

When tagging, always use the exact control ID format from the taxonomy:
- HIPAA: `164.312(a)(1)`, `164.308(a)(5)(ii)(D)`
- SOC 2: `CC6.1`, `P4.2`
- ISO 27001: `A.5.15`, `A.8.10`
- GDPR: `art-17`, `art-5-1-c`
- CCPA: `1798.105`, `gpc`
- CMIA: `56.103`
- ca-misc: `sb568:22581`

Inventing control IDs is the single most damaging mistake you can make in compliance tagging. Always check the taxonomy file first.

## 4. Hard rules — never violate these

These are the spec invariants. Breaking any of them breaks downstream tooling.

1. **The frontmatter schema is a contract.** Required fields are required forever. Optional fields can be left out, but if you include them they must conform.
2. **The intent taxonomy is closed.** Pick from `schemas/intents.v1.yaml` or pick nothing. Never invent tag IDs or compliance control IDs.
3. **Compliance tags require evidence + controls.** No exceptions. Conditional JSON Schema validation enforces it.
4. **Locked files are off-limits.** Any analysis with `lock: true` in its frontmatter must never be edited. Any path listed in `LOCKED.md` must never be edited. If asked to edit one, refuse and explain how to unlock it.
5. **Disabled files are queryable but excluded from analysis passes.** A node with `disabled: true` stays in the manifest but is skipped by `audit`, `compliance`, `coverage`, `drift`. Do not re-analyze disabled files unless explicitly asked.
6. **Audit, contracts, drift, contract-diff read analyses, NOT source code.** This is by design — it forces reliance on the structured contracts and keeps token cost low. Resist the temptation to re-read source.
7. **Never invent dependencies.** When resolving `imports_internal` to source paths, only emit edges that resolve to a real file in the manifest. Unresolved imports stay in `depends_on_raw`.
8. **The manifest is the database. Never hand-edit it.** It's regenerated by `mhng-repo-mind manifest`. If you need to change a node, change the source `.md`'s frontmatter and rebuild.
9. **Compliance output is not legal advice.** Every report you produce in this category must include or preserve the disclaimer that compliance tags are AI-generated structured claims a human compliance officer must review.
10. **Under-tagging is safer than over-tagging.** When in doubt, drop the tag. False positives in compliance reports destroy trust faster than missing tags do.

## 5. The CLI command catalog

If the CLI is installed (Mode 1), these are the commands you can run via Bash. Ones marked **(LLM)** delegate to a Claude Code subagent and cost tokens; others are pure Node and free.

### Bootstrap & sync (LLM)
```bash
mhng-repo-mind init                    # full bootstrap
mhng-repo-mind init --estimate         # token + cost ballpark, no LLM
mhng-repo-mind init --limit N          # cap file count (validate prompt on a small slice)
mhng-repo-mind init --groups a,b,c     # bootstrap only specific groups
mhng-repo-mind init --kinds m,n        # only specific resolved kinds
mhng-repo-mind init --since <git-ref>  # only files changed since ref
mhng-repo-mind sync                    # incremental refresh of stale + spec-stale files
mhng-repo-mind sync --dry-run          # show plan without invoking LLM
mhng-repo-mind add <files...>          # analyze specific files (CI use case)
```

### Deterministic graph operations (no LLM)
```bash
mhng-repo-mind status                  # stale / new / deleted / spec-stale / locked
mhng-repo-mind validate                # validate manifest + frontmatter against JSON Schemas
mhng-repo-mind manifest                # rebuild manifest.json from analyses
mhng-repo-mind symbols                 # SYMBOLS.md reverse index
mhng-repo-mind glossary                # GLOSSARY.md from type-kind nodes
mhng-repo-mind entry-points            # ENTRY_POINTS.md (empty depended_on_by, non-test)
mhng-repo-mind dead-code               # DEAD_CODE.md candidates
mhng-repo-mind graph                   # GRAPH.md Mermaid diagrams per group
mhng-repo-mind intents                 # INTENTS.md grouped by tag
mhng-repo-mind compliance              # COMPLIANCE.md + per-framework files
mhng-repo-mind compliance --csv        # also emit compliance.csv for GRC tools
mhng-repo-mind compliance --gaps       # exit non-zero on missing required_controls
mhng-repo-mind compliance --framework gdpr   # single framework
mhng-repo-mind coverage                # evaluate intent_rules + required_controls
mhng-repo-mind triage                  # create/update AUDIT.triage.md
mhng-repo-mind props                   # extract property-test seeds from enforces
mhng-repo-mind lint                    # run user's typecheck/lint/test, write LINT.md
mhng-repo-mind sources                 # SOURCES.md + sources.json from lockfiles
mhng-repo-mind sources-diff [prev]     # compare lockfile state vs prior capture
mhng-repo-mind exposures               # EXPOSURES.md
mhng-repo-mind freshness               # check declared source freshness windows
mhng-repo-mind snapshot <name>         # capture point-in-time state
mhng-repo-mind snapshot-diff <a> <b>   # auditor-grade diff between two snapshots
mhng-repo-mind test-coverage           # cross-reference enforces vs test imports
mhng-repo-mind provenance              # PROVENANCE.md (who/what/when generated each)
mhng-repo-mind locked                  # LOCKED.md
mhng-repo-mind deprecated              # DEPRECATED.md (past + upcoming)
mhng-repo-mind access-check            # verify private/protected access modifiers
mhng-repo-mind contract-diff <ref>     # compare enforces vs snapshot/git ref
```

### LLM passes (LLM)
```bash
mhng-repo-mind audit                   # cross-file adversarial bug sweep, analyses only
mhng-repo-mind contracts               # match assume/enforce pairs across files
mhng-repo-mind drift <git-ref>         # downstream contract check vs git ref
mhng-repo-mind guide "<question>"      # task-oriented narrative walkthrough
mhng-repo-mind overview                # OVERVIEW.md project synthesis
mhng-repo-mind process                 # generate multi-file deep-dives for declared business processes
mhng-repo-mind process --discover      # propose processes by clustering files; writes DISCOVERED.md for human review
mhng-repo-mind migrate --from <model> [--to <model>]   # re-analyze old-model files
```

### Retrieval (no LLM)
```bash
mhng-repo-mind query "<text>"          # keyword retrieval
mhng-repo-mind query --intent <id>     # exact intent tag match
mhng-repo-mind query --control <id>    # exact compliance control match
mhng-repo-mind query --meta key=value  # filter by user-defined meta field
mhng-repo-mind select <expr>           # preview a selector expression (dbt-style)
```

### Selector grammar
```bash
# Basic forms
mhng-repo-mind select 'tag:security-auth'
mhng-repo-mind select 'control:art-17'
mhng-repo-mind select 'kind:module'
mhng-repo-mind select 'group:auth'
mhng-repo-mind select 'path:packages/auth/**'
mhng-repo-mind select 'state:modified'              # files changed since manifest
mhng-repo-mind select 'model:claude-opus-4-6'
mhng-repo-mind select 'lock:true'

# Walks
mhng-repo-mind select '+packages/auth/src/api.ts'   # api.ts and upstream
mhng-repo-mind select 'packages/auth/src/api.ts+'   # api.ts and downstream
mhng-repo-mind select 'packages/auth/src/api.ts+2'  # downstream depth 2

# Composition
mhng-repo-mind select 'tag:pii&kind:module'         # intersect (AND)
mhng-repo-mind select 'control:art-17,control:1798.105'  # union (OR)
mhng-repo-mind select 'state:modified+'             # modified plus closure

# Indirect selection — auto-include tests
mhng-repo-mind select 'kind:module' --indirect-selection eager     # default
mhng-repo-mind select 'kind:module' --indirect-selection cautious  # only if all deps included
mhng-repo-mind select 'kind:module' --indirect-selection none      # no auto-tests
```

### Doctor / setup
```bash
mhng-repo-mind doctor                  # preflight: node, git, claude, paths, schemas
mhng-repo-mind install-hooks           # git post-commit/post-merge hooks
mhng-repo-mind watch                   # fs.watch → sync on change
mhng-repo-mind runs                    # show recent run history
mhng-repo-mind runs --last             # full JSON of last run
```

### Global flags
- `--config <path>` — path to docs.config.json
- `--verbose` — more logging
- `--json` — machine-readable output
- `--dry-run` — show plan without LLM
- `--force` — overwrite existing files where supported

### Environment variables
- `REPO_MIND_LLM_CMD` — override the LLM runner (default: `claude`)
- `REPO_MIND_LOG_LEVEL` — `debug | info | warn | error`
- `REPO_MIND_PRICE_INPUT` / `REPO_MIND_PRICE_OUTPUT` — USD per 1M tokens for `--estimate`
- `ANTHROPIC_API_KEY` — forwarded to the LLM runner

## 6. Workflow recipes

These are the canonical patterns for common requests. Follow them.

### Recipe A: User asks "document my codebase"

This is the cold-start case. The user has nothing yet.

1. Run `mhng-repo-mind doctor` to verify the environment.
2. If the user hasn't created `docs.config.json`, help them. Copy `reference/docs.config.example.json` and adjust:
   - `target_repo`: the path to the codebase to analyze
   - `output_dir`: where the analyses live (NEVER inside the target repo for deployed apps — see security note in Recipe N)
   - `include` / `exclude`: globs matching the project layout
   - `groups`: how to roll files up into logical packages
   - `kinds`: how to classify file types
3. Run `mhng-repo-mind init --estimate` first. Print the cost ballpark to the user. Wait for confirmation before spending tokens.
4. Run `mhng-repo-mind init --limit 5` first to validate prompt quality on a small slice. Open one of the resulting `.md` files and check that the analysis looks good.
5. If the output looks good, run `mhng-repo-mind init` for real.
6. After init completes, run the standard derived views in this order:
   ```bash
   mhng-repo-mind manifest        # rebuild deterministically (catches any drift)
   mhng-repo-mind overview        # OVERVIEW.md
   mhng-repo-mind compliance      # COMPLIANCE.md + per-framework
   mhng-repo-mind intents         # INTENTS.md
   mhng-repo-mind glossary        # GLOSSARY.md
   mhng-repo-mind entry-points    # ENTRY_POINTS.md
   mhng-repo-mind dead-code       # DEAD_CODE.md
   mhng-repo-mind sources         # SOURCES.md from lockfiles
   ```
7. Tell the user what was produced and link to OVERVIEW.md as the entry point.

### Recipe B: User asks "what kind of project is this"

```bash
mhng-repo-mind overview            # generate OVERVIEW.md if not present
```

Then read OVERVIEW.md and summarize. The first section ("What is this?") names the project type with the most specific accurate label. Don't re-derive — OVERVIEW.md already did the work.

### Recipe C: User asks "find bugs in my code"

```bash
mhng-repo-mind audit               # cross-file adversarial sweep
mhng-repo-mind contracts           # assume/enforce satisfaction check
mhng-repo-mind contract-diff main  # if comparing against a baseline
```

Then read AUDIT.md, CONTRACTS.md, and any per-group `_AUDIT.md` files. Surface the highest-severity findings with citations to the analyses. Never invent findings — every claim must trace to a structured field in the manifest or a quoted analysis.

### Recipe D: User asks "are we compliant with X"

```bash
mhng-repo-mind compliance --framework <framework>
```

Then read `COMPLIANCE-<framework>.md`. The structure is:
1. Summary (mapped vs unmapped controls)
2. Required controls gap check
3. Controls with implementations (with file citations + evidence quotes)
4. Controls with no visible implementation (the gap analysis)

When summarizing for the user:
- Lead with the gap count, not the coverage count. Auditors care about what's missing.
- Quote the evidence strings verbatim. Don't paraphrase — auditors want the exact line.
- Always include the not-legal-advice disclaimer.
- Suggest `mhng-repo-mind coverage` if the user has declared `required_controls` and you want to enforce them in CI.

### Recipe E: User asks "where in the codebase do we handle X"

This is the retrieval case. Don't re-analyze.

1. Try `mhng-repo-mind query "X"` first — keyword retrieval over the manifest.
2. If that doesn't help, think about what intent or compliance control X maps to:
   - "auth / login / sessions" → `mhng-repo-mind query --intent security-auth`
   - "PII / personal data" → `mhng-repo-mind query --intent pii`
   - "deletion / right to be forgotten" → `mhng-repo-mind query --control art-17` or `--control 1798.105`
   - "audit logging" → `mhng-repo-mind query --intent security-audit`
3. If still nothing, read the relevant `_SUMMARY.md` file for the most likely group.
4. Only as a last resort, read individual per-file analyses.

The goal is to answer in 1-2 reads, not by scanning the whole tree.

### Recipe F: User asks "what changed recently and what does it break"

```bash
mhng-repo-mind drift <git-ref>     # PR-style downstream contract check
```

If they want the supply-chain version:
```bash
mhng-repo-mind sources-diff        # what packages upgraded
```

If they want compliance-aware change tracking:
```bash
mhng-repo-mind contract-diff main
mhng-repo-mind snapshot-diff <last-snapshot> <current>
```

Read the resulting `DRIFT.md`, `SOURCES-DIFF.md`, `CONTRACT-DIFF.md`, or `SNAPSHOT-DIFF-*.md` and surface broken contracts, downgraded packages, or removed enforces.

### Recipe G: User asks "explain how X works across files"

```bash
mhng-repo-mind guide "<question>"
```

This generates a `GUIDES/<slug>.md` task-oriented narrative that traces behavior across multiple files, citing analyses. Read the result and present it to the user. If the question is specific enough, the guide will name the exact files involved and the contract chain.

### Recipe H: User asks "what's our supply chain"

```bash
mhng-repo-mind sources             # SOURCES.md + sources.json
mhng-repo-mind freshness           # check declared source review dates
```

Read `SOURCES.md`. The output is grouped by ecosystem (npm, pypi, cargo, go, etc.) with version, license, direct/transitive flag, and lockfile origin per package.

### Recipe I: User asks "I changed the prompt — what do I need to re-analyze"

This is the spec-stale case (spec v1.6).

```bash
mhng-repo-mind status              # shows spec-stale files
mhng-repo-mind sync                # refreshes them (preserves locked + disabled)
```

`spec-stale` means: the source SHA matches what's in the manifest, but the prompt/schema/taxonomy hash that produced the analysis has changed. The analysis is technically out of date even though the file isn't. `sync` handles both.

### Recipe J: User asks "migrate my analyses to a new Claude version"

```bash
mhng-repo-mind provenance                                     # see what models are in use
mhng-repo-mind migrate --from claude-opus-4-6 --to claude-opus-5-0 --dry-run
mhng-repo-mind migrate --from claude-opus-4-6 --to claude-opus-5-0
```

`migrate` finds every node whose `generated_by.model` matches `--from` and queues them for re-analysis. Locked files are skipped. The new analyses carry the new model id automatically.

### Recipe K: User asks "what's about to be deprecated"

```bash
mhng-repo-mind deprecated                          # past + upcoming within 30d
mhng-repo-mind deprecated --warn-days 90           # extend warning window
mhng-repo-mind deprecated --strict                 # CI mode: fail if past deprecation has consumers
```

### Recipe L: User asks "are we violating module boundaries"

```bash
mhng-repo-mind access-check
```

Reports every cross-group import that violates a `private` or `protected` access modifier. Exits non-zero on violations. CI gate.

### Recipe M: User asks "extend mhng-repo-mind itself"

You're about to modify the project, not use it. Different rules apply.

1. Read `analyses/FOR_FUTURE_AI.md` (in the main repo, not the skill bundle) — it's the orientation for AI agents extending mhng-repo-mind.
2. Read `analyses/DECISIONS.md` for the rationale behind every non-obvious choice. Don't undo a decision without understanding why it was made.
3. Read `analyses/ROADMAP.md` for what's already planned vs what was deliberately deferred.
4. Read `analyses/SPEC.md` for the stability surfaces. Schema changes require a spec version bump.
5. Follow the task → file table in `CLAUDE.md` for the file layout.
6. Update `CHANGELOG.md`, `docs/cli.md`, and `analyses/ROADMAP.md` as part of any change.
7. Never break the frontmatter, manifest, or taxonomy schemas without bumping `spec_version`.

### Recipe N: Security — where to put mhng-repo-mind output

For **any deployed app or production codebase**, the analyses contain effectively a complete attack surface map (auth flows, security checks, compliance gaps, suspected bugs by file and line). They must NEVER live inside the target repo.

The correct layout:

```
projects/
├── my-app/                     ← target repo, untouched
└── my-app-docs/                ← separate private git repo
    ├── docs.config.json        ← lives HERE
    └── analyses/               ← all output lands here
```

If the user is about to run `mhng-repo-mind init` on a deployed app and `output_dir` is inside the target repo, **stop them**. Suggest the sibling-directory pattern. Reference `reference/security-deployment.md` for the full guidance.

## 7. When to refuse

Some user requests should be refused even if they're polite. Refuse and explain.

- **"Edit this locked analysis"** — refuse. Tell them to set `lock: false` in the frontmatter and re-run `mhng-repo-mind add <file>`.
- **"Hand-edit `manifest.json`"** — refuse. Tell them to edit the source `.md` frontmatter and run `mhng-repo-mind manifest`.
- **"Add a new compliance control ID I made up"** — refuse. Tell them to add it to `schemas/intents.v1.yaml` first (with a spec version bump) so the validator accepts it.
- **"Tag this file with HIPAA Article 17"** — refuse. HIPAA doesn't have an Article 17, GDPR does. Don't let the user mix frameworks.
- **"Delete this file from the manifest manually"** — refuse. Tell them to remove the source file or set `disabled: true`, then rebuild.
- **"Auto-fix the bugs that audit found"** — refuse. mhng-repo-mind surfaces, never fixes. Auto-fix in security/compliance contexts is dangerous.
- **"Re-run init and overwrite all my hand-tuned analyses"** — refuse unless they explicitly confirm. Check `LOCKED.md` first and warn about every locked file that would be affected.
- **"Mark these compliance gaps as covered without code evidence"** — refuse. The whole point of evidence requirements is preventing this.

## 8. Cost & token awareness

LLM passes cost real money. Be honest with the user about it.

- Before running `init` or `sync` on a large codebase, suggest `--estimate` first.
- A 100-file repo costs roughly $5–15 with Claude Sonnet pricing on a fresh `init`. A `sync` after a small edit costs cents.
- Materialization-aware kinds (`dist`, `generated`, `config`) get short stubs that cost ~10% of a normal analysis. Don't override this without reason.
- The `audit`, `contracts`, and `drift` passes read **analyses, not source**. They're cheap by design — don't change that.
- If a user runs `mhng-repo-mind init` on a 5000-file repo without filters, push back. Suggest `--groups`, `--kinds`, or `--limit` to validate quality before spending the full budget.

## 9. Honest caveats to surface

When using this skill, surface these caveats to the user when relevant:

1. **Compliance reports are AI-generated structured claims, not legal certifications.** A human compliance officer must review them. Auditors will ask about evidence — make sure the `evidence` strings are verbatim quotes, not paraphrases.
2. **The output is sensitive.** AUDIT.md is a partial vulnerability list. COMPLIANCE.md exposes regulatory gaps. Treat the docs repo like internal-only secrets-adjacent material.
3. **The prompts haven't been tuned against every codebase.** First runs on a new project may produce mediocre output. Suggest `--limit 5` to validate before committing to a full run.
4. **mhng-repo-mind doesn't auto-fix anything.** It surfaces; humans decide.
5. **The taxonomy is closed by design.** If a user wants a new compliance framework, that's a spec version bump, not a config tweak.
6. **Locked files are protected for a reason.** The user (or a previous AI session) hand-tuned them. Respect the lock.
7. **`disabled` files stay in the manifest** — they're queryable and findable but excluded from analysis passes. This is intentional and different from deletion.

## 10. The reading order for the documentation store

When you walk into a repo that has a documentation store, read in this order:

1. **`CLAUDE.md`** at the repo root (if it exists) — auto-loaded by Claude Code with the project's invariants
2. **`OVERVIEW.md`** — what kind of project is this
3. **`INDEX.md`** — topic router into everything else
4. **`LOCKED.md`** — what you must not edit
5. **`PROVENANCE.md`** — which model versions produced the existing analyses
6. **`SPEC.md`** in `analyses/` (if extending mhng-repo-mind itself) — the stability contract
7. **`COMPLIANCE.md`** — only if the user's question is compliance-related
8. **Per-file analyses** — only as a last resort, drilling down from a specific question

This is the layered design at work: the high-level files answer most questions cheaply, the per-file analyses are the source of truth you fall back on.

## 11. When in doubt

- Read `analyses/FOR_FUTURE_AI.md` and `analyses/DECISIONS.md` — they contain every "why is it this way" answer the project has accumulated.
- If you're about to do something destructive, stop and ask the user first.
- If you're not sure whether a compliance tag applies, drop it. Under-tagging is safer than over-tagging.
- If the manifest doesn't have what you need, the next step is **`mhng-repo-mind sync`**, not **re-running `init`**.
- If a command seems missing, check `reference/cli-cheatsheet.md` for the full catalog.

---

## Closing — the philosophy

mhng-repo-mind is structured, opinionated, and deliberately constrained. The constraints are what make it queryable, reproducible, and auditable. **Respect the schema. Respect the locks. Respect the taxonomy. Never invent.** When you do that, the documentation store becomes a real engineering artifact — something a compliance officer can hand to an auditor, something a new engineer can read in 30 minutes to onboard, something a future AI session can extend without breaking anything.

That's the whole point.
