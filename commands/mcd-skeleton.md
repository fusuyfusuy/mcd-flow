---
name: mcd-skeleton
description: MCD-Flow Phase 2 — Generate skeleton file tree with CONTRACT pseudocode blocks referencing state transitions. Dispatched by /mcd orchestrator with model sonnet.
---

You are executing **MCD-Flow Phase 2: The Skeleton Builder**.

All file operations happen inside the worktree path provided by the orchestrator.

---

## Your Inputs

- `.mcd/config.json` — read first. Every structural decision below (file extensions, stub body, source directory, comment syntax) comes from `profile.*`.
- `.mcd/manifest.json` — especially the `statechart` field
- `<types_dir>` (from config) — read all files, understand every exported type

---

## Step 1: Plan the File Tree

Use `profile.src_dir` as the root. Subdivide by responsibility:
- One responsibility per file — no god files
- Group related actions into `services/`, boundary code into `routes/` or equivalent, shared low-level utilities into `db<file_ext>`, `logger<file_ext>`, etc. Adapt to language conventions:
  - typescript → `<src_dir>/services/`, `<src_dir>/routes/`, `<src_dir>/middleware/`
  - python → `<src_dir>/services/`, `<src_dir>/api/`, `<src_dir>/middleware/`
  - go → `<src_dir>/services/` (or `internal/services/`), `<src_dir>/handlers/`

**Every `transitions[].action` in the statechart must map to an exported function in a skeleton file.**

---

## Step 2: Generate Skeleton Files

For each file in the planned tree:

1. Write all necessary imports (reference actual types from `<types_dir>`, using idioms from the language)
2. Write function signatures matching the type definitions exactly
3. Inside each function body, write a `<comment_prefix> CONTRACT:` block followed by `<stub_body>` (from config)

**CONTRACT block format** (language-adapted — comment prefix from config):

*typescript:*
```typescript
export async function actionName(input: InputType): Promise<OutputType> {
  // CONTRACT:
  // Transition: <from-state> --[EVENT]--> <to-state>
  // Input: <TypeName> (<types/filename.ts>) — <brief>
  // Output: <TypeName> (<types/filename.ts>) — <brief>
  // Logic:
  //   1. <specific, actionable step — name the function calls>
  //   2. <specific, actionable step>
  throw new Error('not implemented')
}
```

*python:*
```python
async def action_name(input: InputType) -> OutputType:
    # CONTRACT:
    # Transition: <from-state> --[EVENT]--> <to-state>
    # Input: <TypeName> (src/types/filename.py) — <brief>
    # Output: <TypeName> (src/types/filename.py) — <brief>
    # Logic:
    #   1. <specific, actionable step>
    #   2. <specific, actionable step>
    raise NotImplementedError
```

*go:*
```go
func ActionName(input InputType) (OutputType, error) {
    // CONTRACT:
    // Transition: <from-state> --[EVENT]--> <to-state>
    // Input: InputType (internal/types/filename.go) — <brief>
    // Output: OutputType (internal/types/filename.go) — <brief>
    // Logic:
    //   1. <specific, actionable step>
    //   2. <specific, actionable step>
    panic("not implemented")
}
```

**Specificity standard for Logic steps:**
- ❌ Too vague: `1. Process the input data`
- ✅ Acceptable: `1. Call InputSchema.parse(input) — raises ValidationError on invalid input`
- ✅ Acceptable: `2. Query db.users.find_by_email(input.email) — if found, raise EmailTakenError`

**Rules:**
- No implementation logic anywhere — only the CONTRACT block and the stub body
- Function signatures must be identical to what the type files define
- Every error handler action (e.g. `handleValidationError`, `handleTimeout`) must have its own stub
- Error handlers include a `Transition:` line pointing to the error state

---

## Step 3: Update `.mcd/registry.json`

Timestamp via `date -u +"%Y-%m-%dT%H:%M:%SZ"`. Add each generated file:

```json
{
  "phase2": { "status": "complete", "timestamp": "<ISO>" },
  "files": {
    "<src_dir>/services/featureService<file_ext>": {
      "status": "skeleton",
      "hash": null,
      "fillAttempts": 0,
      "verifyAttempts": 0
    }
  }
}
```

---

## Step 4: Gate — Skeleton Structure Approval

Print a summary:
```
=== MCD-Flow Phase 2 Complete ===
Files generated: <count>
  <path> — <N> actions
  ...
Statechart actions covered: <N>/<total>
```

Ask: **"Approve skeleton structure to continue to Phase 3 (audit)? [y/n]"**

- **y** → exit cleanly
- **n** → ask what to change, revise skeleton files, re-show summary, ask again

**Override:** If `.mcd/config.json` contains `"skip_skeleton_gate": true`, or if the orchestrator's context block contains `Gate: skip` for this phase, print `Phase 2 gate bypassed (per user override)` and exit without prompting. Phase 3 (Audit) will still catch structural issues, so the override is safe when the user trusts the manifest → skeleton mapping.
