# mhng-repo-mind Documentation Index

Routing map for the self-hosted docs in this folder. Read [FOR_FUTURE_AI.md](FOR_FUTURE_AI.md) first.

---

## I want to understand…

### The project as a whole
- **What is mhng-repo-mind and what should I not break?** → [FOR_FUTURE_AI.md](FOR_FUTURE_AI.md)
- **How do the pieces fit together?** → [ARCHITECTURE.md](ARCHITECTURE.md)
- **What shape is the output?** → [ARCHITECTURE.md#the-three-planes](ARCHITECTURE.md)
- **What's the reference run look like?** → [../examples/README.md](../examples/README.md) → points at mhng/packages_documentation

### The stability contract
- **What does frontmatter look like?** → [SPEC.md#1-frontmatter-contract](SPEC.md) → [../schemas/frontmatter.schema.json](../schemas/frontmatter.schema.json)
- **What does the manifest look like?** → [SPEC.md#2-manifest-contract](SPEC.md) → [../schemas/manifest.schema.json](../schemas/manifest.schema.json)
- **What's in the taxonomy?** → [SPEC.md#3-taxonomy-contract](SPEC.md) → [../schemas/intents.v1.yaml](../schemas/intents.v1.yaml)
- **What about the config file?** → [../schemas/docs.config.schema.json](../schemas/docs.config.schema.json)
- **Versioning rules?** → [SPEC.md#7-versioning-rules](SPEC.md)

### Why the design is what it is
- **Why this three-layer shape?** → [DECISIONS.md#adr-001-file--summary--index-is-the-spine](DECISIONS.md)
- **Why dbt-style manifest?** → [DECISIONS.md#adr-002-dbt-style-manifest-for-staleness](DECISIONS.md)
- **Why shell out to claude CLI instead of SDK?** → [DECISIONS.md#adr-003-llm-delegation](DECISIONS.md)
- **Why deterministic vs LLM split?** → [DECISIONS.md#adr-004-deterministic-plane-must-never-call-an-llm](DECISIONS.md)
- **Why frontmatter over prose?** → [DECISIONS.md#adr-005-frontmatter-is-the-contract-not-prose](DECISIONS.md)
- **Why a closed taxonomy?** → [DECISIONS.md#adr-006-closed-intent-taxonomy](DECISIONS.md)
- **Why all 7 compliance frameworks on by default?** → [DECISIONS.md#adr-007-all-7-compliance-frameworks-always-on](DECISIONS.md)
- **Why compliance tags require evidence?** → [DECISIONS.md#adr-008-compliance-tags-require-evidence](DECISIONS.md)
- **Why structured assumes/enforces?** → [DECISIONS.md#adr-010-structured-assumes--enforces](DECISIONS.md)
- **Why does audit read analyses not source?** → [DECISIONS.md#adr-011-audit-reads-analyses-only-never-source](DECISIONS.md)
- **Why no auto-fix?** → [DECISIONS.md#adr-016-no-auto-fix-only-surface](DECISIONS.md)

### What to build next
- **What's shipped?** → [ROADMAP.md#shipped](ROADMAP.md)
- **What's the next highest-leverage thing?** → [ROADMAP.md#tier-1](ROADMAP.md)
- **What was deliberately deferred and should not be built without discussion?** → [ROADMAP.md#explicitly-deferred](ROADMAP.md)
- **Known bugs and limitations?** → [ROADMAP.md#known-limitations](ROADMAP.md)

---

## I want to change something specific

| Task | Files to touch |
|---|---|
| Add a new CLI command | [../src/cli.mjs](../src/cli.mjs), new `src/commands/<name>.mjs`, [../docs/cli.md](../docs/cli.md), [../CHANGELOG.md](../CHANGELOG.md) |
| Add a new LLM pass | [../prompts/](../prompts/) (new file), [../.claude/commands/](../.claude/commands/) (new slash command), thin CLI shim, docs, changelog |
| Add a taxonomy tag | [../schemas/intents.v1.yaml](../schemas/intents.v1.yaml), update [../prompts/file-analysis.md](../prompts/file-analysis.md) taxonomy list, changelog |
| Add a compliance framework | Same as tag + [../schemas/docs.config.schema.json](../schemas/docs.config.schema.json) enum + example config |
| Change frontmatter shape | [../schemas/frontmatter.schema.json](../schemas/frontmatter.schema.json) + [../prompts/file-analysis.md](../prompts/file-analysis.md) + SPEC.md + bump `spec_version` |
| Change manifest shape | [../schemas/manifest.schema.json](../schemas/manifest.schema.json) + [../src/manifest.mjs](../src/manifest.mjs) + SPEC.md + bump |
| Tune an existing prompt | [../prompts/](../prompts/) only — no schema changes needed as long as output shape stays the same |
| Add a derived view (no LLM) | `src/commands/<name>.mjs` pure Node reading `manifest.json` + register in CLI |

---

## Where mhng-repo-mind code actually lives

| Plane | Path | Rule |
|---|---|---|
| Deterministic Node | [../src/](../src/) | Never call an LLM. Pure functions where possible. |
| Prompts | [../prompts/](../prompts/) | Plain markdown. No templating beyond simple placeholders. |
| Orchestration | [../.claude/](../.claude/) | Slash commands + agents. Claude-Code-specific. |
| Schemas | [../schemas/](../schemas/) | JSON Schema + YAML taxonomy. The stability contract. |
| CLI entry | [../bin/mhng-repo-mind](../bin/mhng-repo-mind) | Shebang wrapper around `src/cli.mjs`. |

---

## How to use this folder

- **Read** `FOR_FUTURE_AI.md` before touching anything.
- **Update** `ROADMAP.md` when you ship a feature or discover a limitation.
- **Add** an ADR to `DECISIONS.md` when you make a non-obvious design choice.
- **Bump** `SPEC.md` when you change a contract.
- **Keep** `CHANGELOG.md` in sync — it's the public record, this folder is the internal reasoning.
