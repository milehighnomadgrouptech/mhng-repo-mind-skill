# Group summary prompt (spec v1)

You synthesize a `_SUMMARY.md` for one logical group (a package, app, or layer) by reading every per-file analysis `.md` already produced for that group.

## Inputs

- `GROUP_NAME`: name of the group
- `GROUP_FILES`: list of per-file analysis paths in this group
- `OUTPUT_PATH`: where to write `_SUMMARY.md`

## Output format

```markdown
# <GROUP_NAME> — Summary

> Read this first (≈2k tokens). For deep dives, follow the links to per-file analyses.

## Purpose & role in the system
<2–4 sentences>

## Public API surface
<What this group exports for external consumers, with concrete symbol names. Link to the analysis files where they live.>

## Architecture
<How the source modules, configs, tests, and dist artifacts fit together as logical sub-groups.>

## Key modules walkthrough
<For each significant source module: 2–4 sentences on responsibility, key exports, dependencies, invariants. One bullet per module, with a markdown link to its analysis .md.>

## Tests
<What is covered, what is not. Link to test analyses.>

## Build & distribution
<Build config, dist outputs, entry points.>

## Dependencies
<Third-party + internal, with the consumers that justify them.>

## Open questions / gaps
<Anything the per-file analyses flagged as unknown.>
```

## Rules

- **Quote real identifiers, file paths, and function names** drawn from the per-file analyses. Never invent.
- Use markdown links to source `.md` files where helpful (relative paths from this `_SUMMARY.md`).
- If a section has nothing to say for this group, write "Not applicable" rather than padding.
- Lead with the high-signal stuff; assume the reader will stop reading after 30 seconds.
