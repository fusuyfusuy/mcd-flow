# MCD-Flow

**Manifest-Contract-Dispatch** — a structured feature development pipeline for Claude Code that routes work to the right model at each phase, keeping expensive reasoning where it matters and delegating mechanical execution to cheaper models.

## Why it exists

Unstructured AI coding ("vibe coding") produces inconsistent results because the model invents both the architecture and the implementation simultaneously, under the same context pressure. MCD-Flow separates those concerns:

- **Opus** does all design work: requirements, architecture, type contracts, pseudocode, and auditing
- **Sonnet** builds the physical file structure from the approved design
- **Haiku** fills in the implementation, one file at a time, with no design authority

The statechart in the manifest acts as a hard boundary. If implementation logic would violate a defined state transition, it gets caught before any code runs.

## How it works

Six phases, each dispatched as a model-routed sub-agent:

| Phase | Command | Model | Output |
|-------|---------|-------|--------|
| 0 — Discover | `mcd-discover` | Opus | `.mcd/requirements.md` — actors, flows, constraints, error conditions |
| 1 — Plan | `mcd-plan` | Opus | `.mcd/manifest.json`, `.mcd/statechart.mmd`, `types/` |
| 2 — Skeleton | `mcd-skeleton` | Sonnet | `src/` file tree with CONTRACT pseudocode stubs |
| 3 — Audit | `mcd-audit` | Opus | Corrected skeletons, resolved type drift, `tests/` oracle tests |
| 4 — Fill | `mcd-fill` | Haiku | Implemented source files (DAG-ordered, concurrent per level) |
| 5 — Verify | `mcd-verify` | Haiku | Lint + typecheck + test report; orchestrator escalates failures |

**Human review gates** at Phase 0 (requirements), Phase 1 (spec + types), and Phase 3 (skeleton + oracle tests). Nothing proceeds without explicit approval.

## Installation

```bash
claude plugin install /path/to/mcd-flow
```

## Quick start

```
/mcd "user authentication with email and password"
```

That single command runs the full pipeline. Individual phases can be re-run independently:

```
/mcd-discover
/mcd-plan
/mcd-skeleton
/mcd-audit
/mcd-fill
/mcd-verify
```

---

## Tutorial

### Prerequisites

- Claude Code with the mcd-flow plugin installed
- A git repository (`git init` if starting fresh)
- A language toolchain installed for the project you're working in

MCD-Flow is tech-stack agnostic. Phase 1 detects the language from a project marker and writes `.mcd/config.json` declaring the source layout, stub syntax, validation library, test framework, and verification commands. Built-in profiles: **TypeScript** (`package.json` → zod + vitest), **Python** (`pyproject.toml` / `requirements.txt` → pydantic + pytest), **Go** (`go.mod` → validator + `go test`). See `references/language-profiles.md` to add more — no code changes required.

The tutorial below walks through a TypeScript example; the Python and Go flows produce analogous artifacts with their own idioms.

---

### Step 1: Invoke the pipeline

Navigate to your project root and run:

```
/mcd "order processing: users can place orders, each order reserves inventory, payment is charged on fulfillment"
```

MCD-Flow will create an isolated git branch and worktree before touching any files:

```
Created branch:  mcd/order-processing
Created worktree: .worktrees/mcd-order-processing/
```

All work happens inside the worktree. Your main branch is untouched until you approve the final merge.

---

### Step 2: Phase 0 — Requirements (approve or refine)

The Discoverer agent decomposes your prompt and asks about ambiguities before any design is committed:

```
=== MCD-Flow Phase 0 Complete ===
Feature: order-processing
Actors: 2 — Customer, System
Flows: 3 — Place Order, Reserve Inventory, Fulfill Order
Constraints: 4
Error conditions: 6
Out of scope: partial fulfillment, subscription orders

Ambiguities requiring clarification:

1. Should inventory reservation be synchronous (blocking) or async (queue-based)?
   Recommended default: synchronous — simpler, sufficient for stated requirements

2. Is payment idempotent? Can a fulfillment be retried safely?
   Recommended default: yes — use idempotency key on charge call

Approve requirements to continue to Phase 1 (architecture)? [y/n]
```

Answer the ambiguity questions, then approve. If the requirements are wrong, type `n` and describe what to change — the agent revises and re-shows the summary.

**What gets written:** `.mcd/requirements.md`

---

### Step 3: Phase 1 — Architecture (approve the design)

The Architect reads your approved requirements and generates the manifest, statechart, and type definitions:

```
=== MCD-Flow Phase 1 Complete ===
Feature: order-processing
Entities: 3 — Order, InventoryItem, PaymentCharge
Statechart states: 8 — idle, validating, reserving, charging, complete,
                       error.validation, error.inventory, error.payment
Statechart transitions: 7
  SUBMIT: validateOrder (OrderInput → ValidationResult)
  VALID: reserveInventory (ReservationRequest → ReservationResult)
  RESERVED: chargePayment (ChargeRequest → ChargeResult)
  ...
Timeout states: 2 — reserving (10000ms), charging (15000ms)
Statechart diagram: .mcd/statechart.mmd
Type files: types/order.ts, types/inventory.ts, types/payment.ts

Note: Oracle tests are generated in Phase 3 after the skeleton exists.

Approve spec to continue to Phase 2 (skeleton)? [y/n]
```

Review the statechart and types before approving. This is the cheapest point to change the design — every downstream phase derives from this. If a state is missing or a type field is wrong, say `n` and describe the correction.

**What gets written:** `.mcd/manifest.json`, `.mcd/statechart.mmd`, `types/*.ts`

The `.mcd/statechart.mmd` file is a standard Mermaid diagram — open it in any Mermaid viewer to visualize the full state machine before approving.

---

### Step 4: Phase 2 — Skeleton (approve the file tree, or override)

Sonnet reads the manifest and builds the file tree with CONTRACT pseudocode stubs. A gate pauses after generation so you can confirm the structure is right before the more expensive Audit phase runs. Skip the gate with `--no-skeleton-gate` on `/mcd`, or by setting `"skip_skeleton_gate": true` in `.mcd/config.json`:

```typescript
// src/services/orderService.ts

export async function validateOrder(input: OrderInput): Promise<ValidationResult> {
  // CONTRACT:
  // Transition: idle --[SUBMIT]--> validating
  // Input: OrderInput (types/order.ts) — order payload from the client
  // Output: ValidationResult (types/order.ts) — pass/fail with field errors
  // Logic:
  //   1. Call OrderInputSchema.parse(input) — throws ZodError on invalid fields
  //   2. Query db.inventory.findById(input.itemId) — if not found, throw Error('ITEM_NOT_FOUND')
  //   3. Verify input.quantity > 0 — if not, throw Error('INVALID_QUANTITY')
  //   4. Return { valid: true, errors: [] }
  throw new Error('not implemented')
}
```

Every action in the statechart maps to exactly one exported function. Every function body is a numbered recipe — no implementation logic, no guesswork for the Filler.

**What gets written:** `src/**/*.ts` skeleton files, `.mcd/registry.json` updated with `"status": "skeleton"` per file

---

### Step 5: Phase 3 — Audit + Oracle tests (approve the skeleton)

This is the most important gate. Opus reads every skeleton file and runs four checks:

1. **Type reference accuracy** — every `CONTRACT: Input/Output` type actually exists in `types/`
2. **Cross-file consistency** — if `serviceA` calls `serviceB.doThing()`, that function must exist in `serviceB`'s skeleton
3. **Pseudocode specificity** — vague steps like `// handle the data` are rewritten in-place to be implementable
4. **Statechart coverage** — every statechart action has a skeleton function; missing ones are added

After the checks pass, oracle tests are generated from the **actual skeleton file paths** — no guessing at imports:

```
=== MCD-Flow Phase 3 Audit Complete ===
Files audited: 4
Type issues resolved: 1 (PaymentCharge.currency field was missing)
Pseudocode issues resolved: 2 (steps rewritten for specificity)
Statechart coverage: 7/7 actions covered
  Gaps filled: none
  Type files modified: types/payment.ts
Oracle tests generated: 4 test files — 21 tests total
  tests/services/orderService.test.ts — 8 tests (7 transitions + 1 negative case added)
  tests/services/inventoryService.test.ts — 5 tests
  ...
  Happy-path-only functions detected and patched: validateOrder (1 negative case added)

Files:
  src/services/orderService.ts
    validateOrder — idle --[SUBMIT]--> validating
    handleValidationError — validating --[INVALID]--> error.validation
    ...

Approve skeleton to proceed to Phase 4 (code fill)? [y/n]
```

Review the audit report. If the pseudocode for any function looks wrong, type `n`, describe the fix, and the agent will revise and re-show the summary.

**What gets written:** corrected `src/**/*.ts` skeletons, corrected `types/*.ts`, `tests/**/*.test.ts`

---

### Step 6: Phase 4 — Fill (automatic, concurrent)

No gate. Haiku fills each file in dependency order. Files with no mutual dependencies are filled concurrently.

```
Phase 4: Filling 4 files across 3 dependency levels

Level 0 (no dependencies): db.ts
  [filling db.ts...]

Level 1 (depends on Level 0): orderService.ts, inventoryService.ts
  [filling orderService.ts, inventoryService.ts concurrently...]

Level 2 (depends on Level 1): fulfillmentService.ts
  [filling fulfillmentService.ts...]
```

The Filler follows the CONTRACT pseudocode exactly. It cannot change function signatures, cannot add new exports, and removes CONTRACT comments after implementing. The filled file is pure implementation — what was pseudocode is now code.

**What gets written:** fully implemented `src/**/*.ts` files, registry updated to `"status": "filled"`

---

### Step 7: Phase 5 — Verify (automatic, with escalation)

No gate. Haiku runs the `lint`, `typecheck`, and `test` commands declared in `.mcd/config.json` per file, then the orchestrator handles failures:

```
=== MCD-Flow Phase 5 Report ===
Verified: 3
Failed:   1

Failed files:
  --- src/services/inventoryService.ts ---
  Step failed: test
  Error:
  FAIL tests/services/inventoryService.test.ts
    reserveInventory — reserving → error.inventory
      AssertionError: expected { code: 'RESERVATION_FAILED' } to have property 'retryable'
  ---
```

**Escalation ladder (automatic):**

- **Haiku attempt 1:** self-correction with error log fed back
- **Haiku attempt 2:** second self-correction
- **Attempt 3:** pipeline pauses, shows you the diff and error

```
=== MCD-Flow: Verification Failed ===
File: src/services/inventoryService.ts
Attempts: 3

Retry this file with Sonnet (claude-sonnet-4-6)? [y/n/skip]
```

Approve Sonnet, and if that fails, it asks about Opus. If you skip a file, it's marked in the registry and the pipeline continues.

---

### Step 8: Completion

When all files verify:

```
=== MCD-Flow Pipeline Complete ===
Branch:   mcd/order-processing
Worktree: .worktrees/mcd-order-processing/

All tests pass. Merge mcd/order-processing → main? [y/n]
```

- **y** — merges with `--no-ff`, removes the worktree and branch, done
- **n** — branch and worktree are preserved; you get the `gh pr create` command to open a PR

---

### Resuming an interrupted pipeline

If a pipeline is interrupted mid-run, invoking the same feature prompt again detects the existing worktree:

```
Found existing MCD-Flow pipeline for 'order-processing'.
Last completed phase: 3 (Audit)
Resume from Phase 4 (Fill)? [y/n]
```

Approve to skip all completed phases and continue from where it left off.

---

### Adding to an existing feature (diff mode)

MCD-Flow detects an existing `.mcd/manifest.json` and enters diff mode automatically:

```
/mcd "add order cancellation with refund"
```

Diff mode rules:
- Additive only by default — new states, transitions, and types are added alongside existing ones
- Existing states, transitions, and type signatures are never modified without explicit confirmation
- Breaking changes (rename, remove, modify a referenced type) stop and ask before proceeding
- Manifest version is bumped: patch for additive changes, minor if any existing type file is modified
- Only new/changed transitions get new skeleton stubs and tests — nothing is regenerated

---

### Project layout after a completed pipeline

```
your-project/
  .mcd/
    manifest.json        # versioned spec — source of truth for the feature
    statechart.mmd       # Mermaid state diagram — open in any Mermaid viewer
    registry.json        # phase completion + file status + content hashes
    requirements.md      # Phase 0 output — actors, flows, constraints
  types/
    order.ts             # Zod schemas + TypeScript interfaces
    inventory.ts
    payment.ts
  src/
    services/
      orderService.ts    # implemented
      inventoryService.ts
      fulfillmentService.ts
    db.ts
  tests/
    services/
      orderService.test.ts    # oracle tests — generated from actual skeleton paths
      inventoryService.test.ts
      fulfillmentService.test.ts
```

---

### Cost profile

| Phase | Model | Token profile |
|-------|-------|---------------|
| Discover | Opus | Small — prompt + requirements doc |
| Plan | Opus | Medium — manifest + types. Cached for all downstream phases |
| Skeleton | Sonnet | Medium — reads manifest + types, writes stubs |
| Audit | Opus | Large — reads all skeletons. Most expensive single phase |
| Fill | Haiku | Per-file delta only — manifest + relevant types cached as prefix |
| Verify | Haiku | Minimal — runs commands, reads output |

The fill phase is where the cost saving is realized. The manifest and type library are cached as a prompt prefix; each Haiku agent pays only for the individual file being filled, not the full project context.
