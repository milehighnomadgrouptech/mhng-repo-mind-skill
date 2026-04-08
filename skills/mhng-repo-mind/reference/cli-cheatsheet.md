# mhng-repo-mind CLI cheatsheet

## Setup
| Command | Purpose |
|---|---|
| `mhng-repo-mind doctor` | Preflight: node, git, claude, config, target/output paths, schemas |
| `mhng-repo-mind version` | Print CLI version |
| `mhng-repo-mind help` | Print help |

## One-shot (LLM, unattended or semi-attended)
| Command | Purpose |
|---|---|
| `mhng-repo-mind auto` | Estimate cost, prompt, then run the full pipeline against `./docs.config.json` |
| `mhng-repo-mind auto ../some-repo` | Bootstrap config in target if missing, estimate, prompt, run |
| `mhng-repo-mind auto --yes` | Skip the cost-confirmation prompt (still shows the estimate) |
| `mhng-repo-mind auto --skip-estimate` | Skip the estimate + prompt entirely (power-user path) |
| `mhng-repo-mind auto --skip-audit` | Skip the slow LLM audit pass |
| `mhng-repo-mind auto --skip-process` | Skip process discover + generate |
| `mhng-repo-mind auto --output ./mydocs` | Override output_dir on bootstrap |

**Default flow:** `auto` now runs `init --estimate` first to show the cost, then prompts `Continue with the full pipeline against this estimate? [y/N]` (defaults to NO). Pass `--yes` to silence the prompt. When stdin isn't a TTY (CI, background scripts) it proceeds with a warning instead of prompting.

**Pipeline order:** `doctor → init --estimate → (prompt) → init → manifest → validate → symbols → glossary → entry-points → dead-code → graph → intents → compliance → coverage → sources → overview → contracts → audit → process --discover → process`. `doctor`, `init`, `manifest`, and `validate` are critical — a **non-zero exit code** on any of them aborts the pipeline. Everything after is best-effort.

**Metric probes vs exit codes.** Each step also has an optional metric probe (e.g. "how many per-file analyses ended up on disk?"). A metric mismatch does **not** abort — `auto` logs a loud `⚠` and continues to downstream deterministic steps. The probe is a sanity check, not a contract; a false negative must not cost the user a second token bill. The summary table uses `✓` (exit 0, metric OK), `⚠` (exit 0, metric off), `✗` (non-zero exit), and `∅ — skipped (flag)` for anything removed by `--skip-audit` / `--skip-process`. If `init` exits 0 but the metric looks wrong, inspect `output_dir` manually and re-run `mhng-repo-mind sync` to fill any gaps.

**Zero-friction install (from the main repo):**
```bash
# macOS / Linux
git clone https://github.com/milehighnomadgrouptech/mhng-repo-mind.git
cd mhng-repo-mind
./quickstart.sh ../my-target-repo         # runs auto end-to-end with preflight checks
```
```powershell
# Windows PowerShell
git clone https://github.com/milehighnomadgrouptech/mhng-repo-mind.git
cd mhng-repo-mind
.\quickstart.ps1 ..\my-target-repo
```
The quickstart script verifies `node`, `claude`, and `ANTHROPIC_API_KEY`, runs `npm install` on first use, then calls `mhng-repo-mind auto`. No `npm link`, no manual config file.

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

## Benchmark (same question, raw vs mhng-repo-mind)
| Command | Purpose |
|---|---|
| `mhng-repo-mind bench "<question>"` | Side-by-side: raw source vs mhng-repo-mind output for the same question |
| `mhng-repo-mind bench "<q>" --runner sdk` | Force the Anthropic SDK runner (exact token counts, requires `ANTHROPIC_API_KEY`) |
| `mhng-repo-mind bench "<q>" --runner claude-code` | Force the `claude -p` runner (estimated token counts) |
| `mhng-repo-mind bench "<q>" --model claude-opus-4-6` | Override model |
| `mhng-repo-mind bench "<q>" --max-context-chars 600000` | Cap each context (≈150k tokens) |
| `mhng-repo-mind bench "<q>" --only raw` | Run only the raw-repo side |
| `mhng-repo-mind bench "<q>" --only analyses` | Run only the mhng-repo-mind side |
| `mhng-repo-mind bench "<q>" --no-write` | stdout summary only, skip the bench/<slug>.md file |
| `mhng-repo-mind bench "<q>" --json` | Machine-readable output |

Produces stdout summary table + `<output_dir>/bench/<slug>.md` with both answers, token counts, cost, wall time, and truncation flags. Use the SDK runner (the default when `ANTHROPIC_API_KEY` is set) for any numbers you intend to publish — the `claude-code` fallback estimates tokens as `character_count / 4`, which is a rough proxy, not real tokenization.

**Windows note.** `bench --runner claude-code` pipes the prompt via stdin (not argv), so multi-hundred-KB prompts no longer hit the cmd.exe argv length limit. If you see `exit null` from bench on a pre-fix build, update mhng-repo-mind. The runner also honors `REPO_MIND_LLM_CMD` for custom binaries / tests.

Pricing defaults to Opus (`$15/1M in`, `$75/1M out`). Override with `REPO_MIND_PRICE_INPUT` / `REPO_MIND_PRICE_OUTPUT` if you're benching a different model.

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

## Branch parity (spec v1.8)
| Command | Purpose |
|---|---|
| `mhng-repo-mind branch-status` | Show which branch the docs store is aligned with; flag if target HEAD ≠ `manifest.metadata.git_ref` |
| `mhng-repo-mind branch-sync` | Switch the docs store to a sibling branch matching the target's current branch |
| `mhng-repo-mind branch-sync --create` | Bootstrap a new docs branch if one doesn't exist |
| `mhng-repo-mind ask "<q>"` | Retrieval against the manifest |
| `mhng-repo-mind ask "<q>" --branch <name>` | Pin retrieval to a specific docs branch |
| `mhng-repo-mind resolve-merge <file>` | Git merge driver for analysis `.md` files (uses frontmatter `source_sha`); see `branches:` config for install lines |
| `mhng-repo-mind watch --branch-mode` | Watcher follows branch switches in the target repo and re-syncs the matching docs branch |

## Automation
| Command | Purpose |
|---|---|
| `mhng-repo-mind install-hooks` | Install git hooks that run `sync` after commits |
| `mhng-repo-mind watch` | Long-running watcher; runs `sync` on file change |

## Global flags
| Flag | Effect |
|---|---|
| `--config <path>` | Path to docs.config.json (default `./docs.config.json`) |
| `--ref <sha>` | Pin the command to a specific git ref. If target HEAD ≠ `manifest.metadata.git_ref` and `--ref` doesn't reconcile them, the command refuses to run. |
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

## Windows + Node 24 notes

These are the rough edges surfaced by the first real end-to-end run on Windows 11 / Node 24 and fixed in the current mhng-repo-mind release. If you encounter any of them, make sure you're on the post-fix build.

- **`auto` aborts with `metric: 0/<N> analyses present` after init succeeds.** Pre-fix bug — the metric check counted manifest nodes instead of real `.md` files on disk. Post-fix behavior: the metric counts files on disk, and even a real metric mismatch only logs `⚠` and continues.
- **`bench --runner claude-code` dies with `exit null`.** Pre-fix bug — the prompt was passed as an argv element and exceeded the Windows argv limit. Post-fix: prompt is piped via stdin, errors surface stderr + signal.
- **`DEP0190` deprecation warnings on every LLM command.** Pre-fix bug — `spawn(cmd, argsArray, { shell: true })` is deprecated on Node 24 and will be a hard error on Node 26. Post-fix: every offender builds the command line as a single string on Windows.
- **`manifest` reports `<N> nodes, 1 summaries` after init wrote 3.** Pre-fix bug — the builder only synthesized summary entries from `rolls_up_to`. Post-fix: walks `output_dir` for `_SUMMARY.md` files and merges them in.
- **`doctor` warns about sibling clones sharing a parent directory.** Pre-fix bug — the worktree check used `git rev-parse --git-common-dir` and flagged any two repos under a common ancestor. Post-fix: checks nested `output_dir`/`target_repo` via plain path comparison.
- **Every analysis ships with `prompt_sha: "unknown"`.** Pre-fix bug — `init` did not hash the prompt file. Post-fix: the CLI computes the 16-hex sha of `prompts/file-analysis.md` and injects it into the prompt context block; every `generated_by.prompt_sha` is populated on the first run.
- **Frontmatter leaks `src\lib\foo.ts.md`.** Pre-fix bug — `path.join` on Windows. Post-fix: `walkSources` normalizes to forward slashes and `manifest.mjs` uses `path.posix.join` for `analysis` paths.
- **`init --estimate --verbose`** now prints the fully-resolved file list so you can sanity-check scope on large repos without re-running `select`.
