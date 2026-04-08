# Changelog

All notable changes to the **mhng-repo-mind Claude Skill** bundle. This changelog tracks **skill releases** only — for the main project changelog, see [mhng/mhng-repo-mind](https://github.com/milehighnomadgrouptech/mhng-repo-mind/blob/main/CHANGELOG.md).

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

### Added
- Initial standalone skill repo split out from the main project
- `LICENSE` (FSL-1.1-MIT, copyright Logan Banks — Mile High Nomad Group)
- User-facing `README.md` with quickstart and skill-specific guidance
- Detailed `INSTALL.md` with platform-specific instructions
- This `CHANGELOG.md`

### Bundled content (mirrored from mhng/mhng-repo-mind)
- `SKILL.md` — entry point with YAML frontmatter for Claude Code
- `prompts/` — full set of forensic-analysis, group-summary, index-builder, overview, audit-group, contracts-check, drift-check, and guide prompts, plus shared `_partials/`
- `schemas/` — frontmatter, manifest, docs.config schemas + intents.v1.yaml taxonomy with ~140 compliance control IDs
- `analyses/` — self-hosted architecture, decisions, spec, roadmap, and index docs
- `reference/` — quickstart, cli-cheatsheet, security-deployment, example config

### Spec version
This release bundles **spec v1.6** of the mhng-repo-mind project.

---

## Notes

- Skill releases are independent from CLI releases. Skill versioning starts at v0.1.0 with this initial split.
- Each release captures a snapshot of the prompts, schemas, and analyses from the main project as of release time.
- Prompt edits in the main project do not automatically propagate to the skill — releases are cut manually until the project has enough usage to justify automation.
- The mirror is one-way: changes happen in the main project first, then get cut into a skill release.
