# Spec (v1.2) — the stability contract

This file describes the fields, shapes, and invariants that downstream tools rely on. **Breaking any of these requires a major version bump.**

Current spec version: **1.2** (draft, unstable until `v1.0.0`).

## 1. Frontmatter contract

Every per-file analysis `.md` starts with YAML frontmatter conforming to [`schemas/frontmatter.schema.json`](../schemas/frontmatter.schema.json). Required fields:

| Field | Type | Required | Stability |
|---|---|---|---|
| `spec_version` | string (`"1"`) | yes | MUST match taxonomy version |
| `source_file` | string | yes | stable |
| `analysis_path` | string | yes | stable |
| `kind` | string | yes | enum from `docs.config.json.kinds` |
| `source_sha` | string (16 hex chars) | yes | sha256 prefix |

Optional but standardized (breaking changes = minor bump pre-1.0, major post-1.0):

| Field | Type | Purpose |
|---|---|---|
| `group`, `layer` | string | grouping & rollup |
| `exports`, `imports_internal`, `imports_external`, `related` | string[] | graph construction |
| `rolls_up_to` | string | path to the `_SUMMARY.md` this file belongs to |
| `source_mtime`, `analyzed_at` | ISO 8601 | drift detection |
| `assumes`, `enforces`, `risks` | object[] | cross-file contract checker |
| `top_suspicion` | string | bug-hunting forcing function |
| `intents` | object[] | why-does-this-code-exist classification |

### Intent entries

Each `intents[]` entry:

```yaml
- id: <string, must exist in schemas/intents.v1.yaml>
  confidence: high | medium | low
  scope: file | section | block
  locations: [<line range or symbol name>]
  evidence: <one sentence>       # REQUIRED on compliance tags
  controls: [<string>]            # REQUIRED on compliance tags
```

Conditional rule (enforced by JSON Schema `allOf/if/then`):
- If `id` starts with `compliance-`, then both `evidence` and `controls` are required.

## 2. Manifest contract

`manifest.json` conforms to [`schemas/manifest.schema.json`](../schemas/manifest.schema.json). Top-level:

```json
{
  "spec_version": "1",
  "generated_at": "<ISO>",
  "root_repo": "<relative path>",
  "root_analyses": "<relative path>",
  "node_count": <int>,
  "summary_count": <int>,
  "nodes": { "<source_file>": { ... } },
  "summaries": { "<summary_path>": { ... } }
}
```

Each node:

```json
{
  "analysis": "<relative path>",
  "group": "<group-name>",
  "kind": "<kind>",
  "source_sha": "<16 hex>",
  "source_mtime": "<ISO>",
  "analysis_sha": "<16 hex>",
  "exports": ["..."],
  "depends_on": ["<resolved source path>"],
  "depends_on_raw": ["<raw import specifier>"],
  "depended_on_by": ["<source path>"],
  "rolls_up_to": "<summary path>",
  "assumes": [...],
  "enforces": [...],
  "risks": [...],
  "top_suspicion": "...",
  "intents": [...]
}
```

### Graph invariants

1. **`depended_on_by` is the inverse of `depends_on`.** Whenever `depends_on` changes, `depended_on_by` must be recomputed.
2. **`depends_on` only contains source paths that exist as keys in `nodes`.** Unresolved imports stay in `depends_on_raw` — they must never appear in `depends_on`.
3. **Every `rolls_up_to` points at a key in `summaries`.**
4. **`source_sha` is authoritative for staleness.** `source_mtime` is informational only.

## 3. Taxonomy contract

[`schemas/intents.v1.yaml`](../schemas/intents.v1.yaml) is a **closed**, versioned vocabulary. The LLM must never invent IDs — analyses with unknown intent IDs fail validation.

Structure:

```yaml
version: "1"
spec: "1.2"
general:
  <family>:         # correctness | safety | reliability | performance | interface | plumbing
    - id: <kebab-case>
      definition: <one sentence>
compliance:
  <framework>:      # hipaa | soc2 | iso27001 | gdpr | ccpa | cmia | ca-misc
    source_version: <string>
    description: <string>
    legal_disclaimer: true
    controls:
      - id: <framework-native ID>
        short: <label>
        definition: <optional longer text>
        draft: true          # optional
        applies_to: [...]    # optional filter tags
```

### Tag change policy
- **Add** a general intent → minor bump.
- **Add** a compliance control → minor bump + update `source_version` if framework revised.
- **Rename** any ID → major bump and migration notes in `CHANGELOG.md`.
- **Remove** any ID → major bump.
- **Framework version bump** (e.g. ISO 27001:2022 → ISO 27001:2027) → new taxonomy file at `schemas/intents.v2.yaml`; old file kept for backward compat.

## 4. Prompt contracts

Prompts in `prompts/` are versioned implicitly by the spec version in the frontmatter they produce. When a prompt's output shape changes, the frontmatter schema changes too, and both require a spec bump.

**Never change a prompt without also:**
1. Updating `schemas/frontmatter.schema.json` if the output shape changed
2. Updating `CHANGELOG.md`
3. Considering whether `spec_version` should bump

## 4b. Process trees (spec v1.7)

`docs.config.json` may declare `processes: [...]`, an array of independent recursive trees describing user-declared business workflows (e.g. attribution, claims intake, enrollment). Each process is a `processNode` defined in [`schemas/docs.config.schema.json`](../schemas/docs.config.schema.json):

- `name` — stable identifier within its parent. Renaming is a manual operation; never auto-renamed.
- `description` — human prose.
- `stages` — recursive array of child `processNode`s. Empty/missing means this is a leaf.
- `file_patterns` and/or `tags_include` — required on leaves. A leaf's covered files = union of glob matches against the manifest + every file carrying any of the listed intent IDs.
- `lock` — if true, `mhng-repo-mind process` skips regenerating this node's .md.
- `group` (top-level only) — optional INDEX.md grouping label.
- `compliance_relevant` (top-level only) — surfaces the process as a unit in compliance reports.

`mhng-repo-mind process` walks each tree and produces a multi-file deep-dive under `<output_dir>/processes/<process>/`:

- Branches → `<.../>OVERVIEW.md` (rollup).
- Leaves → `<.../><name>.md` (deep-dive on actual source files via per-file analyses).
- Every stage .md carries a `process_node` frontmatter block with breadcrumb, parent/child links, prev/next siblings, and `coverage_sha`.

The manifest's `processes` block (defined in [`schemas/manifest.schema.json`](../schemas/manifest.schema.json)) mirrors the tree shape with computed `coverage_sha` per node. Branches hash over their children; leaves hash over (sorted source `analysis_sha`s + `prompt_sha` + node config). Idempotency: a node regenerates only when its `coverage_sha` changes or `--force` is passed.

Per-file forensic analyses gain an additive `processes: [...]` frontmatter array listing every (process, stage) the file belongs to, so the file's analysis links back to all of them.

**Discovery** (`mhng-repo-mind process --discover`) is read-only against config: it writes `<output_dir>/processes/DISCOVERED.md` with proposed processes for the human to review and ratify into `docs.config.json`. It never edits config and never edits the manifest.

## 5. Config contract

[`schemas/docs.config.schema.json`](../schemas/docs.config.schema.json) defines `docs.config.json`. Required fields: `spec_version`, `target_repo`. Everything else is optional with defaults. When `output_dir` is omitted, it defaults to a sibling directory of `target_repo` named `<basename(target_repo)>-mhng-repo-mind`.

New config fields are additive and don't require a spec bump. Removing or renaming fields does.

## 6. Backwards compatibility promises (post-1.0.0)

Once `v1.0.0` ships:
- A `v1.x` manifest must remain readable by every `v1.x+n` tool.
- Unknown frontmatter fields must be preserved, not stripped.
- Missing optional fields must default to reasonable empty values (`[]`, `""`).
- Analyses written by older minor versions must still validate against the current minor's schema.

Until `v1.0.0`: no promises. Do not promise users stability until the tag ships.

## 7. Versioning rules

- **Spec version** (`spec_version: "1"`) — major. Changes when the data model breaks.
- **Prompt version** — implicit, tied to spec version.
- **CLI version** (`package.json`) — semver, tracks the tool. Can bump without spec changes.
- **Taxonomy version** (`version: "1"` in `intents.v1.yaml`) — independent, but frameworks carry their own `source_version` so downstream tools can detect stale analyses after a framework revision.
