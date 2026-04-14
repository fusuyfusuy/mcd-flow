---
name: mcd-plan
description: MCD-Flow Phase 1 — Detect language, generate SPEC_MANIFEST, statechart (.mmd), and type definitions. Dispatched by /mcd orchestrator with model opus.
---

You are executing **MCD-Flow Phase 1: The Architect**.

All file operations happen inside the worktree path provided by the orchestrator. Do not touch the main workspace.

---

## Your Inputs

- `.mcd/requirements.md` — read this first. All design decisions must be grounded in the requirements. If this file does not exist, proceed from the feature prompt only.

---

## Step 1: Detect Language & Write Config

Read `${CLAUDE_PLUGIN_ROOT}/references/language-profiles.md` to see the built-in profiles and detection rules.

Detect the language from markers in the project root (the directory containing the worktree, not the worktree itself — look at the parent repo):

| Marker | Profile |
|--------|---------|
| `package.json` | typescript |
| `pyproject.toml` or `requirements.txt` | python |
| `go.mod` | go |
| none | **prompt the user** to pick one of the built-in profiles, or describe a custom profile |

If a marker exists but the project's actual toolchain differs (e.g. a Python project using `unittest` rather than `pytest`), ask the user:
```
Detected marker: <marker>
Inferred profile: <language>
Is this correct, or override? [keep / override]
```

Write the chosen profile to `<worktree>/.mcd/config.json` using the exact schema from `references/language-profiles.md`. All later phases rely on this file.

---

## Step 2: Detect Mode

Check if `.mcd/manifest.json` exists in the worktree:
- **Not found** → Greenfield mode: generate everything from scratch.
- **Found** → Diff mode: read existing manifest + scan `<src_dir>`, `<types_dir>`, `<test_dir>` (from config) for current state. Apply the rules below.

### Diff Mode Rules

**Additive by default.** Only extend — do not modify existing entities, transitions, or type signatures unless the prompt explicitly requires it.

| Change type | Rule |
|-------------|------|
| New entity | Add to `entities[]`, add new type file, add new tests |
| New feature / flow | Append to `features[]` and `flows[]`, extend statechart |
| New statechart state or transition | Add alongside existing; do not reorder or rename existing states |
| Modify existing entity field | Require explicit user confirmation before changing a type already referenced in skeleton/filled files |
| Remove entity, field, or transition | Treat as breaking. Stop and confirm with user before proceeding |
| Rename anything | Treat as breaking. Stop and confirm with user before proceeding |

**Version bump:** Increment patch (`1.0.x`) for additive-only. Increment minor (`1.x.0`) if any existing type file is modified.

**Do not regenerate** tests or skeletons for transitions that already exist and are unchanged.

---

## Step 3: Generate `.mcd/manifest.json`

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
- `version`: `"1.0.0"` greenfield; increment patch on diffs
- `entities`: every domain object — fields with name + type, relations with entity + cardinality
- `features`: each capability
- `flows`: step-by-step sequences for each feature
- `dependencies`: external packages needed (name → version range) — use ecosystem-native format from `.mcd/config.json.language`
- `statechart_file`: always `.mcd/statechart.mmd`
- Do not guess implementation details — choose the simpler interpretation when ambiguous. Ambiguities should already be resolved in `requirements.md`.

---

## Step 4: Generate Statechart

Write the statechart both as a human-readable `.mmd` (Mermaid) file and as the machine-readable `statechart` field inside the manifest.

**`.mcd/statechart.mmd`:**
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

Rules:
- Use `stateDiagram-v2`
- Label every transition with `EVENT / actionName`
- Use `after Nms` for timeout transitions
- Every error state must appear explicitly

**`statechart` field inside `manifest.json`:**

Required:
- `initial`: starting state name
- `states`: every state, including all `error.*` substates and at least one `{ "type": "final" }` state
- Every non-final state has at least one outgoing transition
- Every state involving a network/IO operation has an `after` timeout entry
- `transitions`: one object per event — `from`, `event`, `to`, `action`, `input` type name, `output` type name

**Error states are mandatory.** For every happy-path transition, define the corresponding error state and handler action.

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

Every `transitions[].action` becomes an exported function in the skeleton (Phase 2).

---

## Step 5: Generate Type Files

Create files in `<types_dir>` (from `.mcd/config.json`) — one per domain entity or concern. Use `<file_ext>` for filenames.

**Rules:**
- Type definitions + validation schemas only. No logic, no functions.
- Every type referenced in a `transitions[].input` or `transitions[].output` must be exported here.
- Use the `validation_lib` idiom from `.mcd/config.json`.

**Per-language examples:**

*typescript (`validation_lib: zod`):*
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

*python (`validation_lib: pydantic`):*
```python
from datetime import datetime
from pydantic import BaseModel, EmailStr
from uuid import UUID

class User(BaseModel):
    id: UUID
    email: EmailStr
    password_hash: str
    created_at: datetime
```

*go (`validation_lib: go-playground/validator`):*
```go
package types

import "time"

type User struct {
    ID           string    `json:"id" validate:"required,uuid"`
    Email        string    `json:"email" validate:"required,email"`
    PasswordHash string    `json:"password_hash" validate:"required"`
    CreatedAt    time.Time `json:"created_at"`
}
```

For other `validation_lib` values (custom profile), apply the same discipline: express types + runtime validation in idiomatic form, no business logic.

---

## Step 6: Update `.mcd/registry.json`

Timestamp via `date -u +"%Y-%m-%dT%H:%M:%SZ"`:

```json
{
  "phase1": { "status": "complete", "timestamp": "<ISO>" },
  "files": {}
}
```

---

## Step 7: Gate — Spec Approval

Print a structured summary:
```
=== MCD-Flow Phase 1 Complete ===
Feature: <name>
Language: <typescript|python|go|...>
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

Ask: **"Approve spec to continue to Phase 2 (skeleton)? [y/n]"**

- **y** → exit cleanly (orchestrator proceeds to Phase 2)
- **n** → ask what to change, revise manifest/statechart.mmd/types, re-show summary, ask again
