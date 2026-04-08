# Guide generation prompt (spec v1.1)

You produce a **task-oriented narrative** that answers a single user question by tracing behavior across multiple files in the codebase. Guides are how humans actually consume documentation — they don't read per-file analyses end-to-end.

## Inputs

A plan file at `<output_dir>/.mhng-repo-mind-guide-plan.json`:

```json
{
  "question": "<user's question>",
  "slug": "<filesystem slug>",
  "output_path": "GUIDES/<slug>.md",
  "candidates": [
    { "file": "<source path>", "analysis": "<analysis path>", "group": "<group>", "score": <int> }
  ]
}
```

You also have access to `manifest.json` and any candidate analysis `.md` files. **Read the analyses, not the source code.** The analyses are the authoritative pre-digested representation.

## Output format

Write to `output_path`:

```markdown
# <Question, Title Cased>
_Auto-generated guide. Source of truth: the per-file analyses linked below._

## TL;DR
<2–4 sentences answering the question at the highest level. A reader who only reads this section should still walk away with the right mental model.>

## The flow
<Numbered list, 3–10 steps. Each step:
- Names the file/symbol involved (linked to its analysis)
- One sentence on what happens
- Why this step exists (the invariant or contract it's enforcing)>

## Key files
| File | Role | Analysis |
|---|---|---|
| <src path> | <one-line role> | [analysis](<analysis path>) |
...

## Contracts in play
<List the most important `assumes` and `enforces` claims from the candidate analyses that are relevant to this question. This is what makes the guide trustworthy: contracts, not vibes.>

- `<file>` enforces: <invariant>
- `<file>` assumes: <invariant>

## Where to look next
- For deeper detail: <links to the most relevant per-file analyses>
- For related topics: <links to other guides if any exist>
- Open questions surfaced by this guide: <bullets pulled from section 7 of the analyses>
```

## Rules

1. **Cite, don't summarize loosely.** Every claim about what the code does must trace to a specific analysis you can link.
2. **Prefer the candidates the CLI selected.** They were ranked by relevance. If a candidate is irrelevant, drop it from the table — don't pad.
3. **Use the structured contract fields.** The `assumes` / `enforces` blocks in each analysis's frontmatter are the strongest evidence you have. Quote them.
4. **Do not invent flow steps.** If you cannot trace step N → step N+1 from the analyses (via `depends_on` / `depended_on_by`), say "this connection is not visible in the analyses" rather than fabricating a transition.
5. **Length cap**: ≤ 250 lines. Guides are routers, not textbooks. The per-file analyses are the textbook.
6. **No code blocks longer than 10 lines.** If the user wants the code, they'll click through.
7. **End with "open questions"** pulled from section 7 of the candidate analyses. These are the audit gaps the guide could not resolve.
