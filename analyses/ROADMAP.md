# Roadmap

What's done, what's planned, and what was deliberately deferred. Start new work here — if a feature isn't on this list, consider whether it belongs before implementing.

## Shipped (spec v1.2, unreleased)

### Core pipeline
- ✅ Per-file forensic analyses with YAML frontmatter (10-section format)
- ✅ Group summaries (`_SUMMARY.md` per logical group)
- ✅ Repo-level `INDEX.md` (topic router)
- ✅ dbt-style `manifest.json` with `depends_on` / `depended_on_by`
- ✅ Incremental `sync` via source SHA diffing + downstream closure

### CLI commands
- ✅ Base: `init`, `sync`, `add`, `query`, `status`, `validate`, `manifest`, `symbols`, `doctor`, `install-hooks`, `watch`, `version`, `help`
- ✅ Cold-start flags: `--groups`, `--kinds`, `--limit`, `--since`, `--estimate`, `--dry-run`
- ✅ Bug hunting: `audit`, `contracts`, `drift`, `lint`, `triage`, `props`
- ✅ Derived views: `glossary`, `entry-points`, `dead-code`, `graph`
- ✅ Task guides: `guide`
- ✅ Intents & compliance: `intents`, `compliance`, `coverage`, plus `query --intent` / `query --control`

### Schemas
- ✅ `schemas/frontmatter.schema.json` (with conditional validation for compliance tags)
- ✅ `schemas/manifest.schema.json`
- ✅ `schemas/docs.config.schema.json`
- ✅ `schemas/intents.v1.yaml` with 7 compliance frameworks

### Prompts
- ✅ `prompts/file-analysis.md` (10 sections + intents + compliance rules)
- ✅ `prompts/group-summary.md`
- ✅ `prompts/index-builder.md`
- ✅ `prompts/audit-group.md`
- ✅ `prompts/drift-check.md`
- ✅ `prompts/contracts-check.md`
- ✅ `prompts/guide.md`

### Infra
- ✅ GitHub Action template (`.github/workflows/mhng-repo-mind.yml.example`)
- ✅ Self-hosted docs in `analyses/`
- ✅ CHANGELOG, README, CLI docs, CONTRIBUTING, CODE_OF_CONDUCT, LICENSE (MIT)

## Not shipped yet — planned

### Tier 1 — highest leverage, should be next

1. **Real end-to-end run against a live target.** Nothing has actually been executed yet. The first run will reveal bugs in prompts, manifest building, slash command plumbing. This is the single most valuable next action. Recommend: run `mhng-repo-mind init --groups <one-small-group>` against `mhng/packages` to validate the pipeline.

2. **Prompt tuning based on real output.** Once there's output to inspect, tighten `prompts/file-analysis.md` based on what the model over-tags, mis-tags, or under-evidences. This is iterative — expect 3–5 cycles before quality stabilizes.

3. **Deterministic tests.** `node:test` coverage for `hash`, `walk`, `frontmatter`, `manifest`, `stale`, `scope`, `taxonomy`. These modules are pure functions and trivial to test. Do this before the first public release.

4. **`mhng-repo-mind doctor` improvements.** Currently it checks basics; could also check:
   - Claude Code version + API key presence
   - Whether the target repo has at least one file matching the include patterns
   - Whether the taxonomy file parses cleanly

### Tier 2 — useful once the core works

5. **`mhng-repo-mind estimate` as its own command.** Right now it's a flag on `init`. Promote to a top-level command so `mhng-repo-mind estimate --groups auth` works without having to type `init --estimate`.

6. **`mhng-repo-mind reload-taxonomy`.** When the user edits `schemas/intents.v1.yaml`, they currently need to manually re-run `sync` to get the new tags applied. A `reload-taxonomy` command could surface: which files have unknown IDs, which files would benefit from the new IDs, and prompt the user for a targeted `add`.

7. **CI integration docs.** A cookbook section in `docs/cli.md` showing:
   - How to run `mhng-repo-mind drift` as a PR check
   - How to run `mhng-repo-mind coverage --strict` as a merge gate
   - How to publish `COMPLIANCE.md` to GitHub Pages
   - How to export `compliance.csv` into Vanta / Drata / SecureFrame

8. **Interactive triage UI (terminal).** `mhng-repo-mind triage --interactive` walks the user through each `AUDIT.md` finding with keyboard shortcuts: `b` = bug, `f` = false positive, `a` = accepted risk, `w` = wontfix. Writes `AUDIT.triage.md` as it goes.

9. **Language adapters.** Currently the prompt assumes TypeScript/JavaScript as the dominant case. Add worked examples for Python, Go, Rust in `examples/`. Consider per-language prompt variants if the forensic analysis quality varies.

### Tier 3 — experimental, don't build without user demand

10. **Property test generation from `enforces`.** Currently `props` writes seeds. A `--generate` flag could produce runnable fast-check / Hypothesis / proptest files. Experimental because the generated tests will sometimes be wrong in interesting ways.

11. **Historical `CHANGES/` rollups.** Once `drift` is running regularly, aggregate `DRIFT.md` artifacts by month into `CHANGES/<year-month>.md`. Requires a convention for where past DRIFT reports are stored.

12. **IDE plugin.** A VS Code extension that reads `manifest.json` and shows the analysis for the file you're currently editing in a sidebar. Explicitly out of scope for this repo — should be a separate project that consumes the spec.

13. **Multi-target support.** One mhng-repo-mind installation pointing at many target repos, with a top-level `manifest-of-manifests.json`. Only worth building if users ask for it.

14. **Semantic query (LLM).** `query --semantic "how does auth work"` that reads INDEX + manifest + frontmatter via an LLM for cases the deterministic query can't handle. Currently handled by `guide <question>` — may be redundant.

15. **Automatic dead-code PR.** `mhng-repo-mind dead-code --pr` opens a draft PR deleting everything in `DEAD_CODE.md`. Dangerous — requires `--yes-i-know-what-im-doing`.

## Explicitly deferred (do not build without discussion)

- **Web UI.** Plain text + git is the feature. A UI can be a separate project.
- **Database backend.** `manifest.json` is the database.
- **Auto-fix for audit findings.** ADR-016 rules this out. Surface, don't fix.
- **Provider-specific lock-in.** The prompts stay portable. Any change that locks mhng-repo-mind to Claude Code specifically should be questioned.
- **Taxonomy plugin system.** The taxonomy stays closed until v1 stabilizes. Extensibility can come later.
- **Runtime execution of target code.** Static analysis only. Never spawn the target's processes, never execute its tests directly (the `lint` command shells out to user-configured commands, which is different — that's the user's own tooling).

## Known limitations

1. **Never run end-to-end.** The entire pipeline has been scaffolded but not executed against a real target. Expect discovery.
2. **`.claude/commands/docs-*.md` slash commands assume Claude Code.** They won't work with other LLM runners without adapter work.
3. **Prompts are untuned.** Built from first principles, not empirical data. First 2–3 runs will reveal systematic failure modes.
4. **No tests.** The deterministic modules are pure enough that writing tests is easy — it just hasn't been done yet.
5. **Manifest builder doesn't yet validate intents against `taxonomy.mjs`.** Validation happens at `mhng-repo-mind validate` time, but not during `manifest` rebuild. Easy to add: reject unknown IDs as a warning.
6. **`watch` mode uses `fs.watch`**, which is unreliable on some platforms. A `chokidar` fallback would fix this but adds a dep.
7. **The CLI's arg parser is handwritten.** It handles the current commands fine but doesn't do subcommand-specific help. A `yargs`-style parser would be nicer but adds a dep.

## Questions for the next session

If the next AI is reviewing this to plan changes, answer these first:

1. **Has `mhng-repo-mind init` been run against a live target yet?** If yes: what was the output quality? If no: that's the next step.
2. **Are there any new requirements from the user that conflict with the ADRs in [DECISIONS.md](DECISIONS.md)?** If yes: update the ADR with the trade-off before implementing.
3. **Has the taxonomy been tuned based on real tagging results?** If no: do that before adding features.
4. **What does the user actually need next?** If the backlog above isn't aligned with their current goal, the backlog is wrong — update it.
