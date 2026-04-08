# INDEX builder prompt (spec v1)

You produce `INDEX.md` — a topic-based "if you want to understand X, read Y" map for the entire `output_dir`.

## Inputs

- `MANIFEST`: the parsed `manifest.json` (nodes + summaries)
- `SUMMARIES`: list of `_SUMMARY.md` paths

## Output format

```markdown
# <repo name> Documentation Index

A reference map: "if you want to understand X, read Y."

Each leaf `.md` is a forensic per-file analysis. Each `_SUMMARY.md` synthesizes a whole logical group.

---

## Start here (per-group summaries)

| Group | Summary | What it is |
|---|---|---|
| <group> | [<path>](<path>) | <one-line description from the summary> |
| ... | ... | ... |

---

## "I want to understand…"

### <Topic A>
- **<concrete question>** → [<file>](<file>), [<file>](<file>)
- ...

### <Topic B>
- ...

---

## How to use this index
1. **Big picture for one group** → open its `_SUMMARY.md`.
2. **Specific behavior or symbol** → jump from the topic list above to the per-file analysis.
3. **Verify shipped behavior** → cross-check `src/*.md` against the matching `dist/*.md`.
```

## Rules

- Topics should be derived from the actual content of the summaries — don't invent categories.
- Prefer linking directly to the most relevant 1–3 files per question, not dumping every file in the group.
- Every link must resolve to a real file in the manifest.
- Keep total length manageable (≤ ~300 lines). The INDEX is a router, not a textbook.
