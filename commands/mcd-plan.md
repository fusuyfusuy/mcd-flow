---
name: mcd-plan
description: MCD-Flow Phase 1 — Generate SPEC_MANIFEST, statechart (.mmd), and type definitions. Dispatched by /mcd orchestrator with model claude-opus-4-6.
---

You are executing **MCD-Flow Phase 1: The Architect**.

All file operations happen inside the worktree path provided by the orchestrator. Do not touch the main workspace.

---

## Your Inputs

- `.mcd/requirements.md` — read this first. All design decisions must be grounded in the requirements. If this file does not exist, proceed from the feature prompt only.

---

## Step 1: Detect Mode

Check if `.mcd/manifest.json` exists in the worktree:
- **Not found** → Greenfield mode: generate everything from scratch using the feature prompt and requirements.
- **Found** → Diff mode: read existing manifest + scan `src/`, `types/`, `tests/` for current state. Apply the rules below.

### Diff Mode Rules

**Additive by default.** Only extend — do not modify existing entities, transitions, or type signatures unless the prompt explicitly requires it.

| Change type | Rule |
|-------------|------|
| New entity | Add to `entities[]`, add new type file, add new tests |
| New feature / flow | Append to `features[]` and `flows[]`, extend statechart |
| New statechart state or transition | Add alongside existing; do not reorder or rename existing states |
| Modify existing entity field | Require explicit user confirmation before changing a type that is already referenced in skeleton/filled files |
| Remove entity, field, or transition | Treat as breaking. Stop and confirm with user before proceeding |
| Rename anything | Treat as breaking. Stop and confirm with user before proceeding |

**Version bump:** Increment patch (`1.0.x`) for additive-only changes. Increment minor (`1.x.0`) if any existing type file is modified.

**Do not regenerate** tests or skeletons for transitions that already exist and are unchanged.

---

## Step 2: Generate `.mcd/manifest.json`

Write the manifest with these fields:

```json
{
  "version": "1.0.0",
  "feature": "<slugified-feature-name>",
  "created": "<ISO 8601 timestamp>",
  "updated": "<ISO 8601 timestamp>",
  "entities": [...],
  "features": [...],
  "flows": [...],
  "dependencies": {},
  "statechart_file": ".mcd/statechart.mmd",
  "statechart": { ... }
}
```

**Rules:**
- `version`: `"1.0.0"` for greenfield, increment patch for diffs
- `entities`: every domain object — fields with name + type, relations with entity + cardinality
- `features`: each distinct capability with user flows listed as steps
- `flows`: step-by-step sequences for each feature
- `dependencies`: external packages needed (name → semver range)
- `statechart_file`: always `.mcd/statechart.mmd` — points to the Mermaid source file written in Step 2b
- Do not guess at implementation details — choose the simpler interpretation when ambiguous. Ambiguities should have already been resolved in `requirements.md` (Phase 0) — if a constraint is listed there, honour it exactly.

---

## Step 2b: Generate Statechart

Generate the `statechart` field inside `manifest.json` **and** write `.mcd/statechart.mmd` as a standalone Mermaid state diagram. The `.mmd` file is the human-readable, diffable source of truth; the JSON `statechart` field is the machine-readable index used by later phases.

**Write `.mcd/statechart.mmd`:**
```
stateDiagram-v2
  [*] --> idle
  idle --> validating : SUBMIT / validateInput
  validating --> processing : VALID / processFeature
  validating --> error_validation : INVALID / handleValidationError
  validating --> error_timeout : after 5000ms / handleTimeout
  processing --> complete : SUCCESS / finalizeFeature
  processing --> error_processing : FAILURE / handleProcessingError
  error_validation --> idle : RETRY
  error_processing --> idle : RETRY
  error_timeout --> idle : RETRY
  complete --> [*]
```

Rules for the `.mmd` file:
- Use `stateDiagram-v2`
- Label every transition with `EVENT / actionName`
- Use `after Nms` for timeout transitions
- Every error state must appear explicitly

**Generate the `statechart` field inside `manifest.json`.** This is the complete behavioral lifecycle of every feature.

**Required for every statechart:**
- `initial`: the starting state name
- `states`: every state, including all `error.*` substates and at least one `{ "type": "final" }` state
- Every non-final state must have at least one outgoing transition
- Every state involving a network/IO operation must have an `after` timeout entry
- `transitions`: one object per event, each with `from`, `event`, `to`, `action`, `input` type name, `output` type name

**Error states are mandatory.** For every happy-path transition, define the corresponding error state and its handler action.

Example statechart structure:
```json
{
  "initial": "idle",
  "states": {
    "idle": { "on": { "SUBMIT": "validating" } },
    "validating": {
      "on": { "VALID": "processing", "INVALID": "error.validation" },
      "after": { "5000": "error.timeout" }
    },
    "processing": { "on": { "SUCCESS": "complete", "FAILURE": "error.processing" } },
    "complete": { "type": "final" },
    "error": {
      "states": {
        "validation": { "on": { "RETRY": "idle" } },
        "processing": { "on": { "RETRY": "idle" } },
        "timeout": { "on": { "RETRY": "idle" } }
      }
    }
  },
  "transitions": [
    {
      "from": "idle",
      "event": "SUBMIT",
      "to": "validating",
      "action": "validateInput",
      "input": "FeatureInput",
      "output": "ValidationResult"
    }
  ]
}
```

Every `transitions[].action` name becomes an exported function in the skeleton (Phase 2).

---

## Step 3: Generate Type Files

Create files in `types/` — one file per domain entity or concern.

**Rules:**
- Interfaces and Zod schemas only. No logic, no functions.
- Every type referenced in a `transitions[].input` or `transitions[].output` must be exported here.
- Use Zod for runtime validation, TypeScript interfaces for static types.

Example:
```typescript
import { z } from 'zod'

export const UserSchema = z.object({
  id: z.string().uuid(),
  email: z.string().email(),
  passwordHash: z.string(),
  createdAt: z.date(),
})

export type User = z.infer<typeof UserSchema>
```

---

## Step 4: Update `.mcd/registry.json`

Write or update the registry:
```json
{
  "phase1": { "status": "complete", "timestamp": "<ISO now>" },
  "files": {}
}
```

---

## Step 5: Gate — Spec Approval

Print a structured summary:
```
=== MCD-Flow Phase 1 Complete ===
Feature: <name>
Entities: <count> — <list>
Features: <count> — <list>
Statechart states: <count> — <list including error states>
Statechart transitions: <count>
  <event>: <action> (<Input> → <Output>)
  ...
Timeout states: <count> — <list>
Statechart diagram: .mcd/statechart.mmd
Type files: <list>

Note: Oracle tests are generated in Phase 3 (Audit) after the skeleton exists.
```

Then ask: **"Approve spec to continue to Phase 2 (skeleton)? [y/n]"**

- **y** → exit cleanly (orchestrator proceeds to Phase 2)
- **n** → ask what to change, revise manifest/statechart.mmd/types, re-show summary, ask again
