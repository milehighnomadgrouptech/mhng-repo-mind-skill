# For the next AI session — read this first

You are being asked to extend `mhng-repo-mind`, a CLI + prompt library + schema set that turns any codebase into forensic per-file analyses, group summaries, a topic index, and a dbt-style dependency graph, with cross-file bug-hunting, task-oriented guides, and a structured intent + compliance taxonomy.

This file is your 5-minute orientation. It points you at everything else.

## What to read, in order

1. **This file.** The orientation.
2. **[ARCHITECTURE.md](ARCHITECTURE.md)** — what the repo actually is, how the pieces connect, and which files matter.
3. **[SPEC.md](SPEC.md)** — the stability contract. Frontmatter shape, manifest shape, taxonomy. If you break these you break every downstream consumer.
4. **[DECISIONS.md](DECISIONS.md)** — the "why" behind every important design choice. Read before proposing changes.
5. **[ROADMAP.md](ROADMAP.md)** — what's done, what's planned, what was deliberately deferred. Start new work here.
6. **[INDEX.md](INDEX.md)** — topic-based router for everything else in this folder and the source.

## What mhng-repo-mind is (in two sentences)

mhng-repo-mind reads a target codebase, spawns one LLM subagent per source file to produce a structured forensic analysis with YAML frontmatter, then aggregates those analyses into group summaries, a dependency graph (`manifest.json`), a topic index, compliance reports across 7 frameworks, and an adversarial bug audit.

It is **not** a static analyzer — it is a layered documentation + reasoning harness where deterministic code handles the graph and LLM agents handle the prose, with a fixed schema connecting them.

## Rules of engagement for future changes

These are the invariants. Break them and you break the tool.

1. **The frontmatter schema is the contract.** Every per-file analysis starts with YAML frontmatter conforming to `schemas/frontmatter.schema.json`. Downstream commands rely on this. Never change required fields without a spec version bump.

2. **The manifest schema is the other contract.** `schemas/manifest.schema.json`. Same rule — field renames require a major version bump.

3. **The intent taxonomy is closed.** `schemas/intents.v1.yaml`. Adding tags is a minor bump; removing or renaming is major. Never let the LLM invent IDs — validation must reject anything outside the taxonomy.

4. **Prompts live in `prompts/` as plain markdown.** Never bury them in agent definitions. Anyone porting mhng-repo-mind to a different LLM must be able to read them without context.

5. **Deterministic commands must not call an LLM.** Anything that can be done with a hash, a glob, or graph traversal must stay LLM-free. This is what keeps the tool fast, reproducible, and cheap.

6. **LLM commands delegate to Claude Code slash commands.** The CLI (`src/commands/*.mjs`) is a thin shim that runs `claude -p "/docs-..."`. The slash commands (`/.claude/commands/docs-*.md`) contain the orchestration logic and prompt references. This separation is deliberate — see [DECISIONS.md](DECISIONS.md#llm-delegation).

7. **Compliance tags carry legal disclaimers and require evidence.** Never relax this. Compliance output is reviewed by humans with liability — the disclaimers and mandatory evidence fields protect them.

8. **Every new command must have both a CLI handler AND a `docs/cli.md` entry AND a `CHANGELOG.md` entry.** These three are how the tool stays documented as it grows.

## Quick mental model

```
Target repo
    ↓
walk.mjs / globs from docs.config.json        ← deterministic
    ↓
LLM subagents (prompts/file-analysis.md)      ← LLM
    ↓
Per-file .md with YAML frontmatter            ← the data plane
    ↓
manifest.mjs builds the graph                 ← deterministic
    ↓
┌─────────────┬──────────────┬──────────────┬────────────────┐
│             │              │              │                │
↓             ↓              ↓              ↓                ↓
derived       bug-hunting    task guides    compliance       retrieval
views         passes         (LLM)          reports          (query)
(no LLM)      (LLM)                         (no LLM)         (no LLM)
│             │                             │
_SUMMARY.md   AUDIT.md                      COMPLIANCE.md
GLOSSARY.md   CONTRACTS.md                  COMPLIANCE-*.md
INDEX.md      DRIFT.md                      COVERAGE.md
ENTRY_POINTS.md                             INTENTS.md
DEAD_CODE.md
GRAPH.md
SYMBOLS.md
```

## Things that will tempt you and should be resisted

- **"Let's add a web UI."** No. Plain text + git is a feature. A UI can be a separate project that reads the manifest.
- **"Let's make the LLM call a database."** No. The manifest is the database.
- **"Let's support only Claude."** No. The prompts live in `prompts/` so anyone can run them through any LLM. `REPO_MIND_LLM_CMD` is the override.
- **"Let's auto-fix the bugs `audit` finds."** No. mhng-repo-mind surfaces issues; humans decide fixes. Auto-fix is a different tool.
- **"Let's make the taxonomy extensible via plugins."** Not yet. It's closed for v1. Stability over flexibility until the spec matures.

## Where to start if you want to extend it

| If the user asks you to… | Start here |
|---|---|
| Add a new kind of report | `src/commands/<name>.mjs` + register in `src/cli.mjs` + docs + changelog |
| Add a new LLM pass | `prompts/<name>.md` + `.claude/commands/docs-<name>.md` + thin CLI shim + docs |
| Add a taxonomy tag | `schemas/intents.v1.yaml` + update the prompt's allowed-ID list + changelog |
| Add a compliance framework | Same as above, plus update `schemas/docs.config.schema.json` enum + example config |
| Improve per-file analysis quality | `prompts/file-analysis.md` — but never break frontmatter contract |
| Speed up a deterministic command | `src/*.mjs` pure Node — test by running against the reference fixture |
| Change output file format | Requires a spec version bump in `CHANGELOG.md`; update `SPEC.md` |

## The reference run

`examples/README.md` points at `mhng/packages_documentation/analyses/` — 85 forensic analyses, 5 group summaries, manifest, index, all generated against a real TypeScript monorepo. Use it as the canonical example of what the tool's output looks like when things are working.

## Current status

- **Spec version**: 1.2 (unstable until 1.0.0 tag)
- **CLI commands**: 25+
- **LLM commands**: 4 (`init`, `sync`, `add`, plus delegates for `audit`, `contracts`, `drift`, `guide`)
- **Prompts**: 7 (file-analysis, group-summary, index-builder, audit-group, drift-check, contracts-check, guide)
- **Frameworks supported**: HIPAA, SOC 2, ISO 27001, GDPR, CCPA/CPRA, CMIA, California misc
- **Never actually run end-to-end against a live target by the author yet.** The first real run will reveal bugs — expect them.

## One honest thing

The prompts are good but not great yet. They haven't been tuned against real output. The single most valuable thing the next AI session could do is **run `mhng-repo-mind init` against a small real target, inspect the per-file analyses, and tighten the prompt based on what the model over-tags or under-evidences**. That tuning pass is where the quality jump lives.
