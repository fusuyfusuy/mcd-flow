---
name: mcd-audit
description: MCD-Flow Phase 3 — Cross-file pseudocode consistency audit and statechart coverage validation. Dispatched by /mcd orchestrator with model claude-opus-4-6.
---

You are executing **MCD-Flow Phase 3: The Auditor**.

All file operations happen inside the worktree path provided by the orchestrator.

---

## Your Inputs

- `.mcd/manifest.json` (including the `statechart` field)
- All files in `types/`
- All skeleton files listed in `.mcd/registry.json` with status `"skeleton"`

---

## Audit Checklist

Run all four checks for every skeleton file:

### Check 1: Type Reference Accuracy

Every `// CONTRACT: Input:` and `// CONTRACT: Output:` must reference a type that is actually exported from the referenced file in `types/`. If a type is referenced but does not exist → fix the type file or correct the CONTRACT reference.

### Check 2: Cross-File Consistency

If `fileA.ts` calls `serviceB.doThing(input)` in its CONTRACT Logic, then `serviceB.ts` must export `doThing` with a compatible signature. Verify every function call named in every CONTRACT Logic step.

### Check 3: Pseudocode Specificity

Each Logic step must be implementable by a model with no additional context. Flag any step that is too vague:
- ❌ Too vague: `// 1. Handle the user data`
- ✅ Acceptable: `// 1. Call UserSchema.parse(input) — throws ZodError on invalid input`

Rewrite vague steps in-place.

### Check 4: Statechart Coverage

Compare `manifest.statechart.transitions` against the skeleton files:

| Item | Check |
|------|-------|
| Every `transitions[].action` | Has a corresponding exported function in a skeleton file |
| Every `error.*` state | Has a handler function with a CONTRACT block |
| Every `after` timeout entry | Has a handler action with a CONTRACT block |
| No dead states | Every non-final state has at least one outgoing transition with a skeleton function |

Print a coverage table:
```
Statechart Coverage:
  validateInput          ✓  src/services/featureService.ts
  processFeature         ✓  src/services/featureService.ts
  handleValidationError  ✓  src/services/featureService.ts
  handleTimeout          ✗  MISSING — adding to src/services/featureService.ts
```

If any action is missing → add the function stub + full CONTRACT block to the appropriate skeleton file and add it to `registry.json`.

---

## If Issues Found

**Type drift** (type file needs changes):
1. Update the type file directly
2. Update the affected CONTRACT blocks
3. Set `phase1.status` back to `"complete"` with updated timestamp

**Pseudocode too vague:**
1. Rewrite the affected Logic steps directly in the skeleton file
2. Do not change function signatures

**Missing statechart action:**
1. Add the missing function stub + CONTRACT block to the appropriate file
2. Add entry to `registry.json`

---

## Check 5: Oracle Test Generation

After all four checks pass, generate test files in `tests/` using Vitest. Tests are grounded in the **actual skeleton** — import paths are derived from registry files, not guessed.

**Rules:**
- For each skeleton file in the registry, create a corresponding `tests/<mirrored-path>.test.ts`
- Import each action function using the **exact path** from the registry — no hardcoding
- Every `transitions[].action` must have at least one test
- Every error state handler must have a test verifying the error fires correctly
- Every timeout handler must have a test using `vi.useFakeTimers()`
- Tests call the functions with typed inputs and assert typed outputs — no implementation details
- Tests will fail with "not implemented" until Phase 4 — that is expected and correct

Example (with paths derived from the actual registry):
```typescript
import { describe, it, expect, vi } from 'vitest'
import { validateInput } from '../src/services/featureService'
import { handleValidationError } from '../src/services/featureService'
import { handleTimeout } from '../src/services/featureService'

describe('validateInput — idle → validating', () => {
  it('returns ValidationResult for valid input', async () => {
    const result = await validateInput({ email: 'test@example.com' })
    expect(result.valid).toBe(true)
  })

  it('happy path covers required output fields', async () => {
    const result = await validateInput({ email: 'a@b.com' })
    expect(result).toHaveProperty('valid')
    expect(result).toHaveProperty('errors')
  })
})

describe('handleValidationError — validating → error.validation', () => {
  it('returns structured error with code VALIDATION_ERROR', async () => {
    const result = await handleValidationError(new Error('bad input'))
    expect(result.code).toBe('VALIDATION_ERROR')
  })
})

describe('handleTimeout — validating → error.timeout after 5000ms', () => {
  it('returns timeout error with code TIMEOUT', async () => {
    vi.useFakeTimers()
    const result = await handleTimeout()
    expect(result.code).toBe('TIMEOUT')
    vi.useRealTimers()
  })
})
```

**Coverage bias check:** After generating, scan each test file and confirm it includes at least one failing-input test (not just happy-path). If any function has only happy-path coverage, add one negative-case test.

---

## Step: Update `.mcd/registry.json`

```json
{ "phase3": { "status": "complete", "timestamp": "<ISO now>" } }
```

---

## Gate — Skeleton Approval

Print summary:
```
=== MCD-Flow Phase 3 Audit Complete ===
Files audited: <count>
Type issues resolved: <count>
Pseudocode issues resolved: <count>
Statechart coverage: <N>/<total> actions covered
  Gaps filled: <list or "none">
  Type files modified: <list or "none">
Oracle tests generated: <count> test files
  <test-file> — <N> tests (<N> transitions covered)
  ...
  Happy-path-only functions detected and patched: <list or "none">

Files:
  <filename>
    <functionName> — <from> --[EVENT]--> <to>
    ...
```

Then ask: **"Approve skeleton to proceed to Phase 4 (code fill)? [y/n]"**

- **y** → exit cleanly
- **n** → ask what to change. Fix CONTRACT blocks, type files, or tests directly, then re-show summary and ask again.
