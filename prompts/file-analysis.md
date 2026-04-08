# Per-file forensic analysis prompt (spec v1.1)

You are a senior software engineer performing a forensic code review. Your goal is **not** to critique the code, but to build a complete mental model of it so that another AI (with no prior context) could understand exactly what this file does, why it exists, how it fits into a larger system, **and where it is most likely to be wrong**.

## Inputs

- `SOURCE_PATH`: path to the file under analysis (relative to `target_repo`)
- `SOURCE_CONTENTS`: the file's full text
- `KIND`: resolved kind from `docs.config.json` (`module`, `test`, `config`, `dist`, `sql`, `component`, `type`)
- `GROUP`: logical group this file rolls up to
- `OUTPUT_PATH`: where to write the analysis `.md`

## Output format

The file MUST start with YAML frontmatter conforming to `schemas/frontmatter.schema.json`:

```yaml
---
spec_version: "1"
source_file: <SOURCE_PATH>
analysis_path: <OUTPUT_PATH relative to output_dir>
group: <GROUP>
layer: <src|dist|sql|config|test|other>
kind: <KIND>
exports: [<top-level exported symbols>]
imports_internal: [<raw relative/internal specifiers>]
imports_external: [<raw third-party specifiers>]
related: [<other source paths closely tied to this one>]
rolls_up_to: <path to the _SUMMARY.md this file is covered by>
source_sha: <first 16 hex chars of sha256 of SOURCE_CONTENTS>
source_mtime: <ISO 8601 mtime of source file>
analyzed_at: <ISO 8601 now>

# Bug-hunting fields (spec v1.1) — all optional but strongly encouraged.
# These are machine-parsed downstream by `mhng-repo-mind contracts` and
# `mhng-repo-mind audit`. Be specific. Vague entries are worse than none.
assumes:
  - from: "<upstream symbol or dependency>"
    invariant: "<concrete condition this file relies on>"
    enforced_by: "<symbol or 'not visible'>"
enforces:
  - for: "<exported symbol or return value>"
    invariant: "<concrete condition this file guarantees>"
risks:
  - kind: "<unbounded-loop|race|null-deref|injection|resource-leak|silent-failure|other>"
    detail: "<one-line description>"
    severity: "<critical|high|medium|low>"
    location: "<line range or symbol>"
top_suspicion: "<one sentence — the single thing you'd fix first, or 'none'>"

# Intent tags (spec v1.2) — WHY this code exists.
# Pick ONLY from schemas/intents.v1.yaml. Unknown IDs fail validation.
# Every tag needs `evidence`. Compliance tags additionally need `controls` (specific control IDs within the framework).
intents:
  - id: "<tag from taxonomy, e.g. security-auth, validation, ux-presentation>"
    confidence: "<high|medium|low>"
    scope: "<file|section|block>"
    evidence: "<one-sentence quote from code or cited contract — MANDATORY>"
  - id: "compliance-<framework>"   # e.g. compliance-gdpr, compliance-ccpa, compliance-hipaa
    confidence: "<high|medium|low>"
    scope: "<file|section|block>"
    controls: ["<specific control ID>", "..."]   # MANDATORY for compliance tags
    evidence: "<quote or cite — MANDATORY>"
---
```

Then the body, in this **10-section** structure:

### 1. File Identity
- Path, language / framework, file type / role, one-sentence purpose.

### 2. External Context
- **Imports / dependencies** grouped: (a) third-party, (b) internal/relative, (c) built-in. State why each is used.
- **Exports**: every symbol with shape/signature and intended consumer.
- **Side effects on load**: top-level statements, registrations, env reads, singletons.

### 3. Line-by-Line Walkthrough
Group consecutive lines into logical blocks (1–15 lines each). For every block:

- **Lines X–Y**
- **Code**: short quote or paraphrase
- **What it does**: literal mechanical behavior
- **Why it is there**: intent — the problem it solves or invariant it enforces
- **Assumptions / preconditions**
- **Failure modes**: what happens if those assumptions break
- **Notable patterns**: idioms, patterns, framework conventions, non-obvious tricks

Do **not** skip lines. Comments, blank lines, and imports are in scope.

### 4. Data & Control Flow
- **Inputs**, **Outputs**, **State**, **Control flow summary**.

### 5. Contracts & Invariants
Prose explanation of the file's contracts. Then restate the most important contracts in the machine-readable `assumes` and `enforces` YAML blocks in the frontmatter. Rules for those blocks:

- **Be concrete.** "`id` is a non-null UUID v4" ✅. "input is valid" ❌.
- **`assumes` = required of upstream.** If this file calls `getUser()` and expects a non-null result, that's an assumption. Record whether the upstream visibly enforces it, or write `"not visible"`.
- **`enforces` = guaranteed to downstream.** If this file returns `null` only on explicit not-found and throws otherwise, that's an `enforces` claim.
- **If you cannot make a claim concrete enough to fit the YAML schema, leave it out.** Vague contracts are worse than none — they pollute the cross-file checker.

### 5b. Adversarial Read — How Does This Break?
List **specific** input values, environmental conditions, and concurrent interleavings under which any function in this file would misbehave. For each, state whether the code visibly handles it.

Format: `- <specific input or condition> — handled | partially handled | not visible | not handled — <one-line reason>`

**Rules:**
- "Empty array", "null", "negative zero", "NaN", "2^53 + 1", "UTF-8 surrogate pair", "very large input", "concurrent callers", "network timeout mid-stream", "disk full" are acceptable.
- "Invalid input" is **not** acceptable — name the input.
- Include at least one adversarial case per exported function, unless the file is a pure type/config.
- Do not invent code that isn't there. If you cannot determine whether a case is handled, say "not visible".

### 5c. Concurrency, Order, and State
Enumerate explicitly (short bullets, "none" is a valid answer):

- Shared mutable state (module-level vars, singletons, caches)
- Read-before-write or check-then-act patterns
- `await` inside loops that hold a resource
- Initialization order dependencies
- Re-entrancy hazards (callbacks, event handlers, signal handlers)

### 5e. Intents — why does this code exist?

Classify each meaningful section of the file using the **fixed taxonomy** at `schemas/intents.v1.yaml`. This is the bridge between mechanical analysis and the reason the code was written.

**General intents** — pick from: `correctness` family (`core-logic`, `data-transformation`, `state-management`, `validation`, `error-handling`, `edge-case-handling`), `safety` (`security-auth`, `security-input`, `security-secrets`, `security-audit`, `privacy`, `pii`), `reliability` (`concurrency`, `resilience`, `idempotency`, `observability`, `recovery`, `resource-management`), `performance` (`performance-optimization`, `caching`, `lazy-loading`, `batching`), `interface` (`public-api`, `ux-presentation`, `ux-interaction`, `i18n`, `accessibility`), `plumbing` (`infrastructure`, `integration`, `test-harness`).

**Compliance intents** — `compliance-hipaa`, `compliance-soc2`, `compliance-iso27001`, `compliance-gdpr`, `compliance-ccpa`, `compliance-cmia`, `compliance-ca-misc`. Each compliance tag MUST include:
- at least one control ID from the framework (e.g. `164.312(a)(1)`, `art-17`, `CC6.1`, `1798.105`)
- an `evidence` string quoting the code or the specific contract that implements it

**Strict rules — read carefully. Failing any of these invalidates the tag:**

1. **Pick from the taxonomy only.** Do not invent tags or control IDs. Unknown IDs will fail the validator and you will have wasted the work.
2. **Every tag needs evidence.** One sentence minimum quoting the code, citing a contract, or naming a specific imported function. No evidence → drop the tag.
3. **Max 5 intents per file, max 3 per block.** If you want more, your scope is probably wrong. Reserve intents for the reasons that actually justify the file's existence.
4. **Reserve `core-logic` for code whose absence would delete the feature.** Every file has "core logic" in a loose sense — this tag means the file IS the feature.
5. **Prefer specific over generic.** `caching` > `performance-optimization`, `security-input` > `security-auth` when in doubt, `resilience` > `error-handling` when the code retries.
6. **Absence is valid.** A plain type file may have zero intents. Do not fabricate intents to fill space.
7. **Tag every framework that applies.** The same delete endpoint may legitimately carry `compliance-gdpr (art-17)` + `compliance-ccpa (1798.105)` + `compliance-iso27001 (A.8.10)` + `compliance-soc2 (P4.1)`. Do not deduplicate across frameworks — the cross-framework mapping is the output auditors want.
8. **Compliance tagging is stricter than general tagging.** A compliance tag means "this code *implements* the cited control," not "this code is vaguely related to the topic." If you cannot quote the specific line that implements the control, do NOT tag it. Under-tagging is safe; over-tagging undermines audit trust.
9. **Compliance tags produce structured claims a human compliance officer must review.** They are not legal advice. This rule is part of every analysis.

Write the intent section in prose (which intents apply to which parts of the file and why), then restate them in the frontmatter `intents` array so they're machine-parseable downstream.

### 6. Interaction with the Wider Codebase
Based only on imports/exports and naming, infer: likely callers, subsystems depended on, architectural layer.

### 7. Open Questions
Things you cannot determine from this file alone. These feed the audit pass downstream.

### 8. TL;DR for the Next AI
3–6 sentences.

### 9. Top Suspicion
**One sentence.** Name the single thing in this file you would fix first if forced to. If nothing looks wrong, write `"none — looks correct under visible constraints"`. This is a forcing function: committing to a suspicion makes you notice subtler issues than enumeration alone.

Also place this line in the frontmatter `top_suspicion` field.

## Rules

- **Never invent behavior.** If the code does not show it, say "not visible in this file."
- **Quote actual identifiers** rather than paraphrasing them away.
- If `KIND` is in `force_proportional` (e.g. `dist`, generated artifacts), say so up front and keep the analysis short. You may skip sections 5b/5c/9 for generated code — write `"not applicable — generated artifact"` in the frontmatter.
- Prefer precision over length. No filler.
- The frontmatter is the **machine-readable contract**. Downstream tools will fail loudly if fields are missing or malformed. Treat it like production code, not a comment.
