---
name: mcd-audit
description: MCD-Flow Phase 3 — Cross-file pseudocode consistency audit, statechart coverage validation, and oracle test generation. Dispatched by /mcd orchestrator with model opus.
---

You are executing **MCD-Flow Phase 3: The Auditor**.

All file operations happen inside the worktree path provided by the orchestrator.

---

## Your Inputs

- `.mcd/config.json` — drives test framework and file layout
- `.mcd/manifest.json` (including the `statechart` field)
- All files in `<types_dir>`
- All skeleton files listed in `.mcd/registry.json` with status `"skeleton"`

---

## Audit Checklist (run for every skeleton file)

### Check 1: Type Reference Accuracy

Every `CONTRACT: Input:` and `CONTRACT: Output:` line must reference a type actually exported from the named types file. Missing type → fix the type file or correct the CONTRACT reference.

### Check 2: Cross-File Consistency

If `fileA` calls `serviceB.doThing(input)` in its CONTRACT Logic, `serviceB` must export a compatible `doThing`. Verify every function call named in every CONTRACT Logic step.

### Check 3: Pseudocode Specificity

Each Logic step must be implementable by a weaker model with no additional context. Rewrite vague steps in-place.
- ❌ `1. Handle the user data`
- ✅ `1. Call UserSchema.parse(input) — raises ValidationError on invalid input`

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
  validateInput          ✓  <src_dir>/services/featureService<file_ext>
  processFeature         ✓  <src_dir>/services/featureService<file_ext>
  handleValidationError  ✓  <src_dir>/services/featureService<file_ext>
  handleTimeout          ✗  MISSING — adding to <src_dir>/services/featureService<file_ext>
```

Missing action → add the function stub + full CONTRACT block and register it.

---

## If Issues Found

- **Type drift:** update the type file, update affected CONTRACT blocks, set `phase1.status` to `"complete"` with fresh timestamp
- **Vague pseudocode:** rewrite Logic steps in-place; do not change function signatures
- **Missing statechart action:** add stub + CONTRACT block + registry entry

---

## Check 5: Oracle Test Generation

After checks 1–4 pass, generate test files in `<test_dir>` using `profile.test_framework`. **Tests are grounded in the actual skeleton** — import paths are derived from registry files, never guessed.

**Rules (language-neutral):**
- For each skeleton file in the registry, create a corresponding test file at `<test_dir>/<mirrored-path><test_ext>`
- Import each action function using the **exact path** from the registry
- Every `transitions[].action` has at least one test
- Every error state handler has a test verifying the error fires correctly
- Every timeout handler has a test using the framework's fake-timers equivalent
- Tests call functions with typed inputs and assert typed outputs — no implementation details
- Tests will fail with "not implemented" until Phase 4 — expected and correct

**Framework idioms:**

*vitest (typescript):*
```typescript
import { describe, it, expect, vi } from 'vitest'
import { validateInput, handleValidationError, handleTimeout } from '../src/services/featureService'

describe('validateInput — idle → validating', () => {
  it('returns ValidationResult for valid input', async () => {
    const result = await validateInput({ email: 'test@example.com' })
    expect(result.valid).toBe(true)
  })
})

describe('handleTimeout — validating → error.timeout after 5000ms', () => {
  it('returns timeout error', async () => {
    vi.useFakeTimers()
    const result = await handleTimeout()
    expect(result.code).toBe('TIMEOUT')
    vi.useRealTimers()
  })
})
```

*pytest (python):*
```python
import pytest
from freezegun import freeze_time
from src.services.feature_service import validate_input, handle_validation_error, handle_timeout

class TestValidateInput:
    """validate_input — idle → validating"""
    async def test_valid_input_returns_validation_result(self):
        result = await validate_input({"email": "test@example.com"})
        assert result.valid is True

class TestHandleTimeout:
    """handle_timeout — validating → error.timeout after 5000ms"""
    @freeze_time("2024-01-01")
    async def test_returns_timeout_error(self):
        result = await handle_timeout()
        assert result.code == "TIMEOUT"
```

*go test (go):*
```go
package services_test

import (
    "testing"
    "yourproject/internal/services"
)

func TestValidateInput_IdleToValidating(t *testing.T) {
    result, err := services.ValidateInput(Input{Email: "test@example.com"})
    if err != nil { t.Fatal(err) }
    if !result.Valid { t.Errorf("expected valid=true") }
}
```

**Coverage bias check:** After generating, scan each test file and confirm it includes at least one failing-input test *and* at least one error-path assertion (not just happy-path). If any action function has only happy-path coverage, add one negative-case test.

---

## Step: Update `.mcd/registry.json`

Timestamp via `date -u +"%Y-%m-%dT%H:%M:%SZ"`:
```json
{ "phase3": { "status": "complete", "timestamp": "<ISO>" } }
```

---

## Gate — Skeleton Approval

Print:
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

Ask: **"Approve skeleton to proceed to Phase 4 (code fill)? [y/n]"**

- **y** → exit cleanly
- **n** → ask what to change, revise directly, re-show summary
