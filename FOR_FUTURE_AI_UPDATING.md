# 👋 AI agent — read this before updating the skill bundle

This repo is **not** the source of truth. It is a packaged subset of [`mhng-repo-mind`](https://github.com/milehighnomadgrouptech/mhng-repo-mind) shipped as a Claude Skill bundle. Every code change, schema change, and prompt change happens **upstream** in the main repo and is mirrored down here on release.

If the user asks you to "update the skill" or "sync the skill to the main repo", follow this file.

## Source of truth

- **Main repo**: `mhng-repo-mind` (typically cloned to a sibling directory, e.g. `../mhng-repo-mind` or `../mhng-repo-mind-clone`).
- **This repo**: distribution-only. No `src/`, no `.claude/commands/`, no orchestration code.

**Rule**: never edit schemas, prompts, SPEC.md, or CHANGELOG.md directly here. Edit them upstream first, then mirror down.

## Mirror table

| Source (main repo) | Target (this repo) | Mode |
|---|---|---|
| `schemas/docs.config.schema.json` | `schemas/docs.config.schema.json` | **verbatim** |
| `schemas/manifest.schema.json` | `schemas/manifest.schema.json` | **verbatim** |
| `schemas/frontmatter.schema.json` | `schemas/frontmatter.schema.json` | **verbatim** |
| `schemas/intents.v1.yaml` | `schemas/intents.v1.yaml` | **verbatim** |
| `prompts/*.md` (every file) | `prompts/*.md` | **verbatim** |
| `analyses/SPEC.md` | `analyses/SPEC.md` | **verbatim** |
| `CHANGELOG.md` | `CHANGELOG.md` | **merge** — preserve the skill-bundle preamble line at the top ("All notable changes to the **mhng-repo-mind Claude Skill** bundle..."), then paste the rest of the upstream changelog beneath it |
| `README.md` | `README.md` | **merge** — add new commands / version notes to the existing structure. Don't restructure. |
| `SKILL.md` | (this repo's `SKILL.md`) | **merge** — add new commands to the LLM/deterministic catalog, match existing tone |
| `docs/cli.md` | `reference/cli-cheatsheet.md` | **merge** — add a row per new command to the existing cheatsheet format |

## Files that must NEVER be mirrored

These don't exist in the skill bundle. Don't create them, don't reference them.

- `src/` — orchestration code lives in the main repo
- `.claude/commands/` — these `.md` files live in the main repo and are read by the runner as **prompt source** (piped to `claude -p` over stdin). They are not slash commands at runtime as of spec v1.8. Do not mirror them into the skill bundle, and do not tell users to copy them into `~/.claude/commands/`.
- `bin/` — CLI shim lives in the main repo
- `package.json`, `node_modules/`, `package-lock.json`
- `docs/` (other than the cheatsheet target above)
- The main repo's `analyses/` directory (other than `SPEC.md`) — those files are forensic analyses *of* mhng-repo-mind, not skill content

## Files that are skill-specific and must be preserved

Don't overwrite these from the main repo. They are unique to the skill bundle.

- `LICENSE`
- `INSTALL.md`
- `SKILL.md` (merge only — see table)
- `reference/quickstart.md`
- `reference/security-deployment.md`
- `FOR_FUTURE_AI_UPDATING.md` (this file)
- The CHANGELOG.md preamble line about the skill bundle

## When to sync

Sync this repo whenever the main repo:
- Cuts a release (any new tag like `v1.7.0`)
- Adds, removes, or modifies any file under `schemas/` or `prompts/`
- Updates `analyses/SPEC.md`
- Adds a new CLI command (the cheatsheet needs an entry)
- Bumps the spec version

You do NOT need to sync for:
- Changes to `src/` only (orchestration is upstream-only)
- Changes to `.claude/commands/` only
- Changes to upstream `analyses/` files other than SPEC.md
- Changes to `docs/` other than `cli.md`

## How to sync

1. **Confirm the source path** with the user. Don't guess. If they have multiple checkouts (`mhng-repo-mind`, `mhng-repo-mind-clone`, etc.), ask which one is the current source of truth.
2. **Read the main repo's CHANGELOG.md** to see what changed since the last sync. The list of changed files there is the work plan.
3. **Walk the mirror table** above. For each row:
   - **verbatim**: `Read` source, `Write` target. Identical content.
   - **merge**: `Read` both, splice in the new content, preserve skill-specific structure.
4. **Sanity checks** before committing:
   - `node -e "JSON.parse(require('fs').readFileSync('schemas/docs.config.schema.json','utf8'));JSON.parse(require('fs').readFileSync('schemas/manifest.schema.json','utf8'));JSON.parse(require('fs').readFileSync('schemas/frontmatter.schema.json','utf8'));console.log('OK')"`
   - `git diff` — review every change. Look especially for accidental wipes of skill-specific preambles (CHANGELOG, README, SKILL.md).
   - Confirm no `src/` or `.claude/` paths showed up in the diff.
5. **Report to the user** before committing:
   - List of files changed and added
   - Confirmation that schemas parse
   - Anything in the source you couldn't find a target equivalent for
6. **Wait for user approval** before `git add` / `commit` / `push`. Never commit a skill sync without confirmation — a bad mirror ships broken schemas to every consumer of the skill.

## Hard rules

1. **Never edit code, schemas, or prompts in this repo first.** Always upstream → here.
2. **Never run `mhng-repo-mind` commands against this repo.** It has no `src/`. Run them against the main repo.
3. **Never delete the skill-specific files** listed above, even if the source repo doesn't have them.
4. **Never commit without approval.** This repo is published; mistakes are public.
5. **Never sync partially.** If the schemas updated, the prompts and SPEC almost certainly need to update too. Walk the whole mirror table or none of it.

## One-line invocation

When the user says "sync the skill", use this prompt against a general-purpose agent (or do it yourself):

> Read `FOR_FUTURE_AI_UPDATING.md` in the skill repo. Walk the mirror table top to bottom against the source repo at `<path>`. Report the diff before committing.
