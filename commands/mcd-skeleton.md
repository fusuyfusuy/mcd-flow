---
name: mcd-skeleton
description: MCD-Flow Phase 2 — Generate skeleton file tree with CONTRACT pseudocode blocks referencing state transitions. Dispatched by /mcd orchestrator with model claude-sonnet-4-6.
---

You are executing **MCD-Flow Phase 2: The Skeleton Builder**.

All file operations happen inside the worktree path provided by the orchestrator.

---

## Your Inputs

- `.mcd/manifest.json` — read this first, including the `statechart` field
- `types/` — read all files, understand every exported type and Zod schema

---

## Step 1: Plan the File Tree

Based on the manifest, determine the physical file structure:
- Detect project type from `manifest.dependencies` and feature names
- TypeScript: use `src/services/`, `src/routes/`, `src/middleware/`, etc. as appropriate
- One responsibility per file — no god files
- **Every `transitions[].action` in the statechart must map to an exported function in a skeleton file**

---

## Step 2: Generate Skeleton Files

For each file in the planned tree:

1. Write all necessary imports (reference actual types from `types/`)
2. Write function signatures matching the type definitions exactly
3. Inside each function body, write a `// CONTRACT:` block followed by `throw new Error('not implemented')`

**CONTRACT block format:**
```typescript
export async function actionName(input: InputType): Promise<OutputType> {
  // CONTRACT:
  // Transition: <from-state> --[EVENT]--> <to-state>
  // Input: <TypeName> (<types/filename.ts>) — <brief description>
  // Output: <TypeName> (<types/filename.ts>) — <brief description>
  // Logic:
  //   1. <specific, actionable step — name the function calls>
  //   2. <specific, actionable step>
  //   3. <specific, actionable step>
  throw new Error('not implemented')
}
```

**Specificity standard for Logic steps:**
- ❌ Too vague: `// 1. Process the input data`
- ✅ Acceptable: `// 1. Call InputSchema.parse(input) — throws ZodError on invalid input`
- ✅ Acceptable: `// 2. Query db.users.findByEmail(input.email) — if found, throw Error('EMAIL_TAKEN')`

**Rules:**
- No implementation logic anywhere — only the CONTRACT block and `throw new Error('not implemented')`
- Function signatures must be identical to what the type files define
- Every error handler action (e.g. `handleValidationError`, `handleTimeout`) must have its own stub
- Error handlers must include the `Transition:` line pointing to the error state

---

## Step 3: Update `.mcd/registry.json`

Add each generated file to the registry and mark phase 2 complete:

```json
{
  "phase2": { "status": "complete", "timestamp": "<ISO now>" },
  "files": {
    "src/services/featureService.ts": {
      "status": "skeleton",
      "hash": null,
      "fillAttempts": 0,
      "verifyAttempts": 0
    }
  }
}
```

Exit cleanly. There is no user gate after Phase 2 — Phase 3 (Audit) immediately reviews and corrects all skeleton output before any code is filled.
