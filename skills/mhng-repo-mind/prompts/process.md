# Process deep-dive prompt (spec v1.7)

You generate a **multi-file deep-dive** for a single user-declared business process. A process is a recursive tree of stages. The CLI has already done the deterministic work: matched source files to stages, computed hashes, and decided which nodes need regeneration. Your job is to write the prose.

## Inputs

A plan file at `<output_dir>/.mhng-repo-mind-process-plan.json`:

```json
{
  "spec_version": "1",
  "generated_at": "<iso>",
  "only_process": null,
  "force": false,
  "processes": [
    {
      "name": "attribution_insurance",
      "description": "...",
      "group": null,
      "compliance_relevant": false,
      "tree": {
        "name": "attribution_insurance",
        "kind": "root",
        "breadcrumb": ["attribution_insurance"],
        "md_path": "processes/attribution_insurance/OVERVIEW.md",
        "source_files": [...],          // union across all leaves
        "coverage_sha": "...",
        "children": [
          {
            "name": "intake",
            "kind": "branch",
            "breadcrumb": ["attribution_insurance","intake"],
            "md_path": "processes/attribution_insurance/intake/OVERVIEW.md",
            "children": [
              {
                "name": "validation",
                "kind": "leaf",
                "breadcrumb": ["attribution_insurance","intake","validation"],
                "md_path": "processes/attribution_insurance/intake/validation.md",
                "source_files": ["models/.../validate.sql", ...],
                "coverage_sha": "..."
              }
            ]
          }
        ]
      }
    }
  ]
}
```

You also have access to `manifest.json` and every per-file forensic analysis under `<output_dir>/`. **Read the analyses, not the source code.** They are the authoritative pre-digested representation.

## Generation order (CRITICAL)

**Bottom-up.** Generate every leaf first, then every branch in dependency order, then the root last. A branch's OVERVIEW must reference its children — those children must already exist on disk before you write the branch.

Within a single process tree:

1. Walk the tree depth-first.
2. At each leaf: generate the leaf .md (deep-dive on actual source files).
3. After all children of a branch exist: generate the branch's OVERVIEW.md.
4. After all top-level branches under the root exist: generate the root's OVERVIEW.md.

## Output formats

### Leaf .md (deep-dive)

```markdown
---
spec_version: "1"
process_node:
  process: <process_name>
  name: <stage_name>
  kind: leaf
  breadcrumb: [<process_name>, ..., <stage_name>]
  parent: <relative path to parent OVERVIEW.md>
  children: []
  prev_sibling: <relative path or null>
  next_sibling: <relative path or null>
  source_files: [<source paths>]
  coverage_sha: <from plan>
generated_by:
  tool: mhng-repo-mind
  tool_version: <semver>
  spec_version: "1"
  model: <model id>
  runner: claude-code
  prompt_path: prompts/process.md
  prompt_sha: <16 hex>
  generated_at: <ISO>
---

# <Stage Name, Title Cased>

> **<process_name> → <branch1> → <branch2> → <stage_name>**
> Parent: [<branch_name>](<relative parent OVERVIEW.md>)
> Prev: [<prev>](...) · Next: [<next>](...)

## Purpose
<2–4 sentences. What this stage does, why it exists, what it would break if removed.>

## Inputs
<Bullet list. Each input is either a source table/file/upstream stage or an external source. Link source files to their forensic analyses.>

## Logic walkthrough
<3–10 numbered steps tracing the stage's behavior. Each step cites a specific analysis section. Do NOT paste source code. Reference symbols by name.>

## Edge cases & failure modes
<Pulled from the `risks` and `top_suspicion` blocks of the covered analyses. One bullet per real risk. If none, write "None surfaced by per-file analyses.">

## Downstream consumers
<List of stages and external consumers that depend on this stage's output. Use `depended_on_by` from the manifest.>

## Source files
| File | Role | Analysis |
|---|---|---|
| <src> | <one-line role> | [analysis](<analysis path>) |

## Open questions
<Pulled from section 7 of the covered analyses. Bullets only. If none, omit the section.>
```

### Branch OVERVIEW.md (rollup)

```markdown
---
spec_version: "1"
process_node:
  process: <process_name>
  name: <branch_name>
  kind: branch
  breadcrumb: [<process_name>, ..., <branch_name>]
  parent: <path to parent OVERVIEW.md or null if direct child of root>
  children: [<paths to child OVERVIEW.md / leaf .md>]
  prev_sibling: <path or null>
  next_sibling: <path or null>
  coverage_sha: <from plan>
generated_by: { ...same shape as leaf... }
---

# <Branch Name>

> **<process_name> → ... → <branch_name>**
> Parent: [<...>](<...>)

## What this stage does
<3–6 sentences. Highest-level summary of the branch's role in the process.>

## Sub-stages
<Numbered list of children, each linked. One sentence per child explaining its role and how it hands off to the next.>

1. [**<child_name>**](<child path>) — <one-line summary>
2. ...

## End-to-end flow through this branch
<A short narrative (≤ 15 lines) walking from the first sub-stage to the last. Describe the data shape entering and exiting the branch.>

## Cross-references
- Parent: [<parent>](<...>)
- Sibling stages: <links>
- External consumers: <if relevant>
```

### Root OVERVIEW.md (process top)

Same as branch OVERVIEW.md plus:

- A "## Why this process exists" section (1–3 paragraphs of business motivation, pulled from the process `description` and the analyses).
- A "## At a glance" table listing every leaf in the tree with its source-file count.
- No `parent` field in `process_node` (set to `null`).

## Hard rules

1. **Never invent stages or rename them.** The plan is authoritative. If the plan says four leaves under `intake`, write four leaves with those exact names.
2. **Never invent source files.** Every source file referenced must come from the leaf's `source_files` array.
3. **Never read source code directly.** Read the per-file forensic analyses. They are the source of truth.
4. **Cite every claim.** Each logic step, each edge case, each risk must trace to an analysis you can link.
5. **Never write to a leaf or branch with `lock: true`.** Skip it silently and proceed.
6. **Write bottom-up.** Children before parents, always. A parent OVERVIEW that links to a child file that doesn't yet exist is a bug.
7. **Length cap**: leaf ≤ 250 lines, branch OVERVIEW ≤ 150 lines, root OVERVIEW ≤ 200 lines. Processes are routers; the analyses are the textbook.
8. **Frontmatter contract.** Every file must conform to `schemas/frontmatter.schema.json`. The `process_node` block is required on every process .md. The `generated_by` block is required on every machine-written file.
9. **Cross-link with relative paths.** Always relative, never absolute. Links should resolve when the docs are viewed in any markdown renderer.
10. **Do not touch files outside `<output_dir>/processes/<process_name>/`.** Per-file forensic analyses are updated separately by `mhng-repo-mind sync`, not by this prompt.
