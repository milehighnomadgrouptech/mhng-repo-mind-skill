# Drift check prompt (spec v1.1)

You are checking whether a change in **one upstream file** has broken the contracts its **downstream consumers** rely on.

## Inputs

- `CHANGED_FILE`: the source file that changed
- `DIFF`: unified diff of the change
- `PREVIOUS_ANALYSIS`: the prior forensic analysis of `CHANGED_FILE` (or "none" if new)
- `NEW_ANALYSIS`: the refreshed forensic analysis of `CHANGED_FILE`
- `DOWNSTREAM_ANALYSES`: list of `{ path, analysis_contents }` for every file in `depended_on_by[CHANGED_FILE]`

## The only question you answer

For **each** downstream file, did the change break a contract that file relies on?

## Output format

One block per downstream file, in this exact shape:

```
<downstream_path>: <VERDICT>
<one line justification>
```

Where `VERDICT` is **exactly one** of:

- `safe` — no contract that the downstream relies on was affected by the change
- `possibly_broken` — a relevant assume/enforce mismatch exists but you cannot confirm without runtime trace; cite the specific `assumes` entry and the upstream field that changed
- `broken` — the change directly violates a downstream `assumes` entry; cite the exact symbol and the assumed invariant

No other verdicts. No prose sections. No "potentially", "might", "could". Commit to one of the three.

## Rules

1. **Base your verdict on the downstream's `assumes` block** matched against the upstream's new `enforces` block and the diff.
2. **Signature changes** (added/removed/renamed/retyped exports) affect every downstream that imports the symbol — mark those `broken` and cite.
3. **Pure additions** (new exports, new optional params with defaults, new files) are `safe` for all downstreams.
4. **Changed return types or thrown exceptions** are `broken` or `possibly_broken` depending on whether the downstream's assume explicitly matches the old behavior.
5. **If the downstream has no `assumes` entries at all**, return `possibly_broken` with the justification `"no contracts declared — cannot verify"`. This surfaces audit gaps without falsely greenlighting.
6. **Do not speculate about bugs unrelated to the change.** Drift-check is scoped narrowly. Other bugs are for `mhng-repo-mind audit`.

## Example

```
packages/auth/src/middleware.ts: broken
session.getUser() return type changed from User to User|null; middleware.ts assumes non-null at line 42.

packages/auth/src/api.test.ts: safe
Test imports unrelated symbols (signin, signout); no assumes entry references getUser.

packages/auth/src/admin-portal.ts: possibly_broken
no contracts declared — cannot verify
```

This is the entire output. No header, no summary, no "overall status".
