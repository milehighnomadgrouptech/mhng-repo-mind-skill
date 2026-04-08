# mhng-repo-mind CLI cheatsheet

## Setup
| Command | Purpose |
|---|---|
| `mhng-repo-mind doctor` | Preflight: node, git, claude, config, target/output paths, schemas |
| `mhng-repo-mind version` | Print CLI version |
| `mhng-repo-mind help` | Print help |

## One-shot (LLM, unattended)
| Command | Purpose |
|---|---|
| `mhng-repo-mind auto` | Run the full pipeline against `./docs.config.json` end-to-end |
| `mhng-repo-mind auto ../some-repo` | Bootstrap config in target if missing, then run the pipeline |
| `mhng-repo-mind auto --skip-audit` | Skip the slow LLM audit pass |
| `mhng-repo-mind auto --skip-process` | Skip process discover + generate |
| `mhng-repo-mind auto --output ./mydocs` | Override output_dir on bootstrap |

**Pipeline order:** `doctor → init → manifest → validate → symbols → glossary → entry-points → dead-code → graph → intents → compliance → coverage → sources → overview → contracts → audit → process --discover → process`. `doctor`, `init`, `manifest` are critical; everything after is best-effort.

**Recommended Claude Code prompt** (paste into a Claude Code session in the target repo):

```
Run `mhng-repo-mind auto` against this repo end-to-end and don't ask me
questions. If docs.config.json doesn't exist, let `auto` bootstrap one.
Use the default pipeline. When it finishes, give me the run summary plus
a one-paragraph review of what's in <output_dir>/OVERVIEW.md.
```

**Recommended settings:**
- Model: `claude-opus-4-6` (forensic analyses + audit benefit from Opus reasoning)
- Permission mode: pre-approved allowlist via `.claude/settings.json`; only use `--dangerously-skip-permissions` in CI / sandboxed containers
- `REPO_MIND_LOG_LEVEL=info` so you can see which step is running
- Run `mhng-repo-mind init --estimate` first to see token/cost scope before committing to a full run

## Bootstrap & sync (LLM)
| Command | Purpose |
|---|---|
| `mhng-repo-mind init` | Full bootstrap of analyses, summaries, manifest, INDEX |
| `mhng-repo-mind init --estimate` | Token + cost ballpark, do not call LLM |
| `mhng-repo-mind init --limit N` | Cap file count (validate prompt on a small slice) |
| `mhng-repo-mind init --groups a,b,c` | Bootstrap only specific groups |
| `mhng-repo-mind init --kinds m,n` | Only specific resolved kinds |
| `mhng-repo-mind init --since <git-ref>` | Only files changed since ref |
| `mhng-repo-mind sync` | Incrementally refresh stale analyses |
| `mhng-repo-mind sync --dry-run` | Show what would change |
| `mhng-repo-mind add <files...>` | Analyze specific files (CI use case) |

## Deterministic (no LLM)
| Command | Purpose |
|---|---|
| `mhng-repo-mind status` | List stale / new / deleted source files |
| `mhng-repo-mind validate` | Validate manifest + frontmatter against JSON Schemas |
| `mhng-repo-mind manifest` | Rebuild manifest.json from current analyses |
| `mhng-repo-mind symbols` | Build SYMBOLS.md reverse index |
| `mhng-repo-mind glossary` | Build GLOSSARY.md from type-kind nodes |
| `mhng-repo-mind entry-points` | Build ENTRY_POINTS.md (empty depended_on_by) |
| `mhng-repo-mind dead-code` | Build DEAD_CODE.md candidates |
| `mhng-repo-mind graph` | Build GRAPH.md Mermaid diagrams |
| `mhng-repo-mind intents` | Build INTENTS.md (why each file exists) |
| `mhng-repo-mind compliance` | Build COMPLIANCE.md + per-framework reports |
| `mhng-repo-mind compliance --csv` | Also emit compliance.csv |
| `mhng-repo-mind compliance --gaps` | Exit non-zero on missing required_controls |
| `mhng-repo-mind compliance --framework gdpr` | Single framework only |
| `mhng-repo-mind coverage` | Evaluate intent_rules + required_controls (CI gate) |
| `mhng-repo-mind triage` | Create/update AUDIT.triage.md from AUDIT.md |
| `mhng-repo-mind props` | Extract property-test seeds from `enforces` claims |
| `mhng-repo-mind lint` | Run user's typecheck/lint/test, write LINT.md baseline |

## Bug hunting (LLM)
| Command | Purpose |
|---|---|
| `mhng-repo-mind audit` | Cross-file adversarial sweep, analyses only (not source) |
| `mhng-repo-mind contracts` | Match assume/enforce pairs across files |
| `mhng-repo-mind drift <ref>` | Check downstream contracts against a git ref |

## Task-oriented (LLM)
| Command | Purpose |
|---|---|
| `mhng-repo-mind overview` | Generate OVERVIEW.md — what kind of project is this |
| `mhng-repo-mind guide "<question>"` | Generate a cross-file narrative guide in GUIDES/ |
| `mhng-repo-mind process` | Generate multi-file deep-dives for user-declared business processes (one OVERVIEW.md per branch, one `<stage>.md` per leaf) |
| `mhng-repo-mind process --discover` | Propose processes by clustering files; writes `DISCOVERED.md` for human review (never edits config) |
| `mhng-repo-mind process --process <name>` | Regenerate a single process tree only |
| `mhng-repo-mind process --force` | Regenerate even unchanged (matching `coverage_sha`) nodes |
| `mhng-repo-mind process --dry-run` | Show which nodes would be regenerated without invoking the LLM |

## Retrieval
| Command | Purpose |
|---|---|
| `mhng-repo-mind query "<text>"` | Keyword retrieval over manifest |
| `mhng-repo-mind query --intent <id>` | Exact intent tag match |
| `mhng-repo-mind query --control <id>` | Exact compliance control match |
| `mhng-repo-mind query --intent compliance-gdpr` | All files tagged GDPR |

## Automation
| Command | Purpose |
|---|---|
| `mhng-repo-mind install-hooks` | Install git hooks that run `sync` after commits |
| `mhng-repo-mind watch` | Long-running watcher; runs `sync` on file change |

## Global flags
| Flag | Effect |
|---|---|
| `--config <path>` | Path to docs.config.json (default `./docs.config.json`) |
| `--verbose` | More logging |
| `--json` | Machine-readable output where applicable |
| `--dry-run` | Show plan without invoking LLM |
| `--force` | Overwrite existing files (where supported) |

## Environment
| Variable | Purpose |
|---|---|
| `REPO_MIND_LLM_CMD` | Override the LLM runner. Default: `claude` |
| `ANTHROPIC_API_KEY` | Forwarded to the LLM runner |
| `REPO_MIND_PRICE_INPUT` | USD per 1M input tokens for `init --estimate` |
| `REPO_MIND_PRICE_OUTPUT` | USD per 1M output tokens for `init --estimate` |
