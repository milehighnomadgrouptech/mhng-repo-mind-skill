# mhng-repo-mind Claude Skill

> The Claude Code skill bundle for [**mhng-repo-mind**](https://github.com/milehighnomadgrouptech/mhng-repo-mind) — drop this folder into `~/.claude/skills/` and use mhng-repo-mind methodology directly inside any Claude Code session, with or without the CLI installed.

mhng-repo-mind turns any codebase into structured AI-generated documentation: forensic per-file analyses, a dbt-style dependency graph, cross-file bug audits, and audit-ready compliance reports with cited control IDs across HIPAA, SOC 2, ISO 27001, GDPR, CCPA, CMIA, and California statutes. Incremental and provenance-tracked.

This bundle is the **Claude-native distribution** of the project. It's self-contained: every prompt, every schema, every doc Claude needs to operate the methodology lives inside this folder.

---

## What you get

After installing the skill, Claude can:

- **Generate forensic per-file analyses** for any codebase with a strict 10-section format and structured YAML frontmatter (exports, dependencies, contracts, intents, top suspicion).
- **Build a `manifest.json` dependency graph** across the codebase.
- **Run compliance checks** against 7 frameworks with ~140 specific control IDs.
- **Find cross-file bugs** by matching `assumes` and `enforces` contracts across the dependency graph.
- **Produce a project overview**, glossary, entry points, dead code report, dependency diagrams, and task-oriented narrative guides.

If you also install the **`mhng-repo-mind` CLI**, Claude shells out to it for the heavy lifting. If not, Claude operates the prompts directly — slower, but it works without any install.

---

## Install

### macOS / Linux
```bash
mkdir -p ~/.claude/skills
git clone https://github.com/milehighnomadgrouptech/mhng-repo-mind-skill.git ~/.claude/skills/mhng-repo-mind
```

### Windows (PowerShell)
```powershell
mkdir -Force "$HOME\.claude\skills"
git clone https://github.com/milehighnomadgrouptech/mhng-repo-mind-skill.git "$HOME\.claude\skills\mhng-repo-mind"
```

### Or download the zip
Grab the latest release from [Releases](https://github.com/milehighnomadgrouptech/mhng-repo-mind-skill/releases) and unzip into `~/.claude/skills/mhng-repo-mind/`.

Once installed, restart Claude Code (or run `/skills` if your version supports it). The skill is now available — Claude will load it automatically when your request matches the skill description (documentation generation, compliance reports, cross-file bug audits, project overviews, etc.).

See [INSTALL.md](INSTALL.md) for detailed instructions and platform-specific notes.

---

## Quickstart

After installing, just ask Claude things like:

> "Generate a project overview for the repo I have open."

> "Run a compliance check against GDPR and tell me which controls have no implementation."

> "Find every file that handles PII and tag it with the relevant CCPA controls."

> "Audit this codebase for cross-file contract bugs."

> "Show me everywhere we implement HIPAA §164.312(a)(1) access control."

> "Update the documentation for the files I changed in this branch."

Claude reads `SKILL.md` to learn how to dispatch each request and follows the prompts in `prompts/` to produce the output.

---

## What's in this bundle

```
mhng-repo-mind-skill/
├── SKILL.md                 ← entry point Claude reads first (YAML frontmatter)
├── README.md                ← user-facing guide (this file)
├── INSTALL.md               ← step-by-step install for Mac / Linux / Windows
├── LICENSE                  ← FSL-1.1-MIT
├── CHANGELOG.md             ← skill release history
├── prompts/                 ← every prompt mhng-repo-mind uses
│   ├── file-analysis.md     (the master 10-section forensic prompt)
│   ├── group-summary.md
│   ├── index-builder.md
│   ├── overview.md
│   ├── audit-group.md
│   ├── contracts-check.md
│   ├── drift-check.md
│   ├── guide.md
│   └── _partials/           (shared rule fragments)
├── schemas/                 ← contracts (JSON Schema + YAML taxonomy)
│   ├── frontmatter.schema.json
│   ├── manifest.schema.json
│   ├── docs.config.schema.json
│   └── intents.v1.yaml      (~140 compliance control IDs across 7 frameworks)
├── analyses/                ← mhng-repo-mind's own self-hosted documentation
│   ├── FOR_FUTURE_AI.md     (start here if you're an AI agent extending it)
│   ├── ARCHITECTURE.md
│   ├── SPEC.md
│   ├── DECISIONS.md
│   ├── ROADMAP.md
│   └── INDEX.md
└── reference/
    ├── quickstart.md
    ├── cli-cheatsheet.md
    ├── security-deployment.md
    └── docs.config.example.json
```

---

## Without the CLI

The skill is **operable without the CLI**. Claude reads `prompts/file-analysis.md` and produces the same forensic per-file analyses by hand, building the manifest as it goes. This is slower than the CLI but it works on any machine that has Claude Code.

For most users, the workflow is:

1. Install the skill (this bundle)
2. Install the [`mhng-repo-mind` CLI](https://github.com/milehighnomadgrouptech/mhng-repo-mind) (optional but recommended)
3. Ask Claude to operate mhng-repo-mind on your codebase

If you want the full CLI experience (incremental sync, scheduled hooks, GitHub Actions integration), grab the main project at [mhng/mhng-repo-mind](https://github.com/milehighnomadgrouptech/mhng-repo-mind).

---

## Compliance frameworks supported

Always on by default:

| Framework | Source | Code-relevant controls |
|---|---|---|
| **HIPAA** | 45 CFR §164 | ~17 |
| **SOC 2** | AICPA Trust Services Criteria 2017 | ~22 |
| **ISO 27001:2022** | Annex A | ~40 |
| **GDPR** | Regulation (EU) 2016/679 | ~33 |
| **CCPA / CPRA** | Cal. Civ. Code §1798.100+ | ~19 |
| **CMIA** | California medical confidentiality | ~6 |
| **California misc** | SB-327, SB-568, AB-375 | ~5 |

Each control is identified by its native ID (`164.312(a)(1)`, `CC6.1`, `art-17`, `1798.105`, etc.) so the output drops straight into a real audit trail.

**This is not legal advice.** Compliance tags are AI-generated structured claims that a human compliance officer must review. Every report ships with that disclaimer.

---

## Security note

mhng-repo-mind output describes your codebase's auth flows, security checks, suspected bugs, and compliance gaps in detail. It's effectively an attack-surface map. Treat the documentation store like internal-only secrets-adjacent material:

- **Private repo**, restricted access
- **Never inside a deployed app**
- **No public CI logs** that print analysis content

See [`reference/security-deployment.md`](reference/security-deployment.md) for the full guidance.

---

## License

Licensed under the [Functional Source License v1.1 with MIT Future](LICENSE) (FSL-1.1-MIT).

Copyright © Logan Banks — Mile High Nomad Group.

You can use, modify, and self-host this skill freely for any non-competing purpose. After two years, each release automatically converts to the standard MIT License.

For commercial licensing or partnerships, contact [logan.banks@mhng.tech](mailto:logan.banks@mhng.tech).

---

## Links

- 🏠 **Main project:** https://github.com/milehighnomadgrouptech/mhng-repo-mind
- 📖 **CLI documentation:** https://github.com/milehighnomadgrouptech/mhng-repo-mind/blob/main/docs/cli.md
- 🐛 **Issues:** https://github.com/milehighnomadgrouptech/mhng-repo-mind-skill/issues
- ✉️ **Commercial:** logan.banks@mhng.tech
