# mhng-repo-mind quickstart

## Install (one-time)

```bash
git clone https://github.com/<your-org>/mhng-repo-mind.git
cd mhng-repo-mind
npm install
npm link    # exposes `mhng-repo-mind` globally
```

## Use against a target repo

The analyses must NOT live inside a deployed app. Use a **separate private documentation repo**.

```bash
# 1. Create a sibling docs repo
mkdir -p ../my-app-docs
cd ../my-app-docs
git init
echo "# Docs (private)" > README.md
echo "node_modules/" > .gitignore

# 2. Configure mhng-repo-mind
cp ../mhng-repo-mind/docs.config.example.json docs.config.json
$EDITOR docs.config.json
# - set "target_repo": "../my-app"
# - set "output_dir": "./analyses"
# - adjust include/exclude/groups for your app's layout

# 3. Preflight
mhng-repo-mind doctor

# 4. Cost estimate (no LLM)
mhng-repo-mind init --estimate

# 5. Validate prompt on a small slice
mhng-repo-mind init --limit 5
# inspect: cat analyses/<some>.md
# if quality is poor, tighten ../mhng-repo-mind/prompts/file-analysis.md before continuing

# 6. Full run
mhng-repo-mind init

# 7. Generate everything else
mhng-repo-mind overview              # OVERVIEW.md — what kind of project is this
mhng-repo-mind compliance            # COMPLIANCE.md + 7 framework files
mhng-repo-mind intents               # INTENTS.md — why each file exists
mhng-repo-mind glossary              # GLOSSARY.md — domain terms
mhng-repo-mind entry-points          # ENTRY_POINTS.md
mhng-repo-mind dead-code             # DEAD_CODE.md candidates
mhng-repo-mind symbols               # SYMBOLS.md reverse index
mhng-repo-mind graph                 # GRAPH.md Mermaid diagrams
mhng-repo-mind audit                 # AUDIT.md cross-file bug findings
mhng-repo-mind contracts             # CONTRACTS.md cross-file contract check
mhng-repo-mind coverage              # COVERAGE.md required-controls + intent rules
```

## Daily workflow

```bash
# After making code changes in the target repo
mhng-repo-mind sync                  # refresh stale analyses incrementally
mhng-repo-mind drift main            # check downstream impact vs main
```

## CI integration

Copy `.github/workflows/mhng-repo-mind.yml.example` from the mhng-repo-mind root into the target repo's `.github/workflows/mhng-repo-mind.yml`. It handles the same-repo and sibling-repo strategies and runs `mhng-repo-mind add <changed-files>` on every push.
