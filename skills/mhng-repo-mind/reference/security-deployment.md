# Security & deployment

## Where to put mhng-repo-mind output

**Never inside a deployed app.** The analyses contain effectively a complete attack surface map: auth flows, security checks, suspected bugs by file and line, compliance gaps, all the structured `assumes` / `enforces` contracts. This is more sensitive than the source code itself, because it concentrates the most exploitable information in one place.

### Layouts ranked

| Layout | Verdict |
|---|---|
| Inside `app/analyses/` (same repo) | ❌ Bundlers can pick up markdown. Anyone with read access sees your bug map. |
| Sibling folder, same git repo | ❌ Same problem. |
| **Sibling folder, separate private git repo** | ✅ This. The only safe choice for deployed apps. |

## Recommended layout

```
projects/
├── my-app/                              ← target repo, untouched
│   └── ...
└── my-app-docs/                         ← separate private repo
    ├── .git/
    ├── .gitignore
    ├── docs.config.json                 ← lives HERE, not in my-app
    └── analyses/
        ├── manifest.json
        ├── INDEX.md
        ├── OVERVIEW.md
        ├── COMPLIANCE.md
        ├── COMPLIANCE-hipaa.md
        ├── ...
        └── packages/
            └── ... mirrored source tree
```

## Belt and suspenders

Even with the docs in a separate repo, add this to the target repo's `.gitignore`:

```
# mhng-repo-mind output should never live here
analyses/
*.mhng-repo-mind*
```

So if anyone runs `mhng-repo-mind init` from the wrong directory, the output gets ignored by git.

## Access control

Treat the documentation repo's contents like internal-only secrets-adjacent material:

- **Private repo**, restricted access matching production secrets
- **No public CI logs** that print analysis content
- **Don't paste analyses into public chat tools or issue trackers**
- Contractors should sign the same NDA that covers production code access
- Same retention/deletion policies as security incident reports

## What's in the docs that's sensitive

- `AUDIT.md` lists suspected bugs by file and line — a partial vulnerability list
- `COMPLIANCE.md` shows exactly which controls you do and don't implement — a regulatory exposure list
- `INTENTS.md` reveals where auth, secrets handling, and PII flows live
- `CONTRACTS.md` enumerates trust boundaries and assumed invariants — exactly what an attacker would map first
- Per-file analyses describe security checks and their bypass conditions

## CI considerations

If you run `mhng-repo-mind drift` or `coverage` in CI:

- Set `ANTHROPIC_API_KEY` as a CI secret, not a plaintext env var
- Scope the job to a specific branch and require manual approval for forks
- Output goes to artifacts, not to PR comments by default (PR comments leak more easily)
- Use the GitHub Action template at `.github/workflows/mhng-repo-mind.yml.example` as a starting point — note its `if: false` gates that you must explicitly enable

## When publishing mhng-repo-mind output is OK

- **Open-source libraries** with no sensitive surface (no auth, no PII, no internal services)
- **Pure utility code** (parsers, math libs, formatters)
- **Public APIs that are already documented elsewhere**

For these, you can ship the analyses inside the same repo and even publish them as a docs site. But this is the exception, not the rule.
