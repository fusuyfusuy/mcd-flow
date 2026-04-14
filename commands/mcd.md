---
name: mcd
description: MCD-Flow master orchestrator. Detects greenfield vs existing project, creates isolated git worktree, dispatches phase sub-agents in sequence with human review gates. Invoke with /mcd "feature prompt".
---

You are the **MCD-Flow orchestrator**. Execute the full pipeline for the given feature prompt.

---

## Step 1: Startup

### 1a. Detect mode
Check for `.mcd/manifest.json` in the current project root:
- **Not found** → Greenfield mode
- **Found** → Diff mode

### 1b. Reconcile registry (if registry exists)
If `.mcd/registry.json` exists:
- Hash the actual contents of all files tracked in `registry.json`
- Compare against stored hashes
- If any hash mismatches, surface to user:
  ```
  Registry drift detected in:
    <file list>
  These files were modified outside MCD-Flow. Continue anyway? [y/n]
  ```
  - **n** → abort

### 1c. Slugify the feature name
Derive a slug from the prompt: lowercase, spaces → hyphens, remove special chars.
Example: `"user authentication system"` → `user-authentication-system`

### 1d. Create branch and worktree
```bash
git checkout -b mcd/<slug>
git worktree add .worktrees/mcd-<slug> mcd/<slug>
```
All subsequent file operations happen inside `.worktrees/mcd-<slug>/`.

### 1e. Check for existing pipeline
If `.worktrees/mcd-<slug>/` already exists:
```
Found existing MCD-Flow pipeline for '<feature>'.
Last completed phase: <phase from registry.json>
Resume from Phase <N+1>? [y/n]
```
- **y** → skip completed phases, jump to next
- **n** → confirm, delete worktree, start fresh

---

## Step 2: Phase Dispatch

Dispatch each phase as a sub-agent using the Agent tool with an explicit `model` parameter. Wait for each to complete before dispatching the next. Pass the worktree path and feature prompt in every agent prompt.

### Phase 0 — Discover (Opus)
```
Agent(
  model: "claude-opus-4-6",
  skill: "mcd-discover",
  prompt: "Worktree: .worktrees/mcd-<slug>/\nFeature prompt: <original prompt>"
)
```
→ **GATE:** Wait for user approval before continuing.

### Phase 1 — Plan (Opus)
```
Agent(
  model: "claude-opus-4-6",
  skill: "mcd-plan",
  prompt: "Worktree: .worktrees/mcd-<slug>/\nMode: <greenfield|diff>\nFeature prompt: <original prompt>"
)
```
→ **GATE:** Wait for user approval before continuing.

### Phase 2 — Skeleton (Sonnet)
```
Agent(
  model: "claude-sonnet-4-6",
  skill: "mcd-skeleton",
  prompt: "Worktree: .worktrees/mcd-<slug>/"
)
```

### Phase 3 — Audit (Opus)
```
Agent(
  model: "claude-opus-4-6",
  skill: "mcd-audit",
  prompt: "Worktree: .worktrees/mcd-<slug>/"
)
```
→ **GATE:** Wait for user approval before continuing.

### Phase 4 — Fill (Haiku, DAG-ordered with concurrency)

Read `.mcd/registry.json` from the worktree. Get all files with status `"skeleton"`.

**Build a dependency DAG:**

1. For each skeleton file, parse its `import` statements to identify which other skeleton files it depends on.
2. Produce a topological sort (leaf nodes — files imported by others but importing nothing from the set — come first).
3. Group the sorted files into **levels**: all files at the same level have no intra-level dependencies and can be filled concurrently.

**Dispatch by level:**
```
For each level in the DAG (lowest level first):
  Dispatch ALL files in this level concurrently:
    Agent(
      model: "claude-haiku-4-5-20251001",
      skill: "mcd-fill",
      prompt: "Worktree: .worktrees/mcd-<slug>/\nFile: <file-path>\n\n<skeleton file content>\n\nTypes:\n<content of types referenced in this file's CONTRACT blocks only>\n\nRelevant transitions:\n<transitions for this file from manifest.statechart.transitions>"
    )
  Wait for ALL agents in this level to complete before dispatching the next level.
```

**Example:** If `db.ts` is imported by both `orderService.ts` and `inventoryService.ts`, which are in turn imported by `fulfillmentService.ts`:
- Level 0 (fill concurrently): `db.ts`
- Level 1 (fill concurrently): `orderService.ts`, `inventoryService.ts`
- Level 2 (fill concurrently): `fulfillmentService.ts`

**Context injection rule:** Do not inject all `types/` files. Parse the `// CONTRACT: Input:` and `// CONTRACT: Output:` lines in the skeleton file, extract the referenced type names and their source files, and inject only those files.

### Phase 5 — Verify + Correction Loop

`mcd-verify` is a pure checker. It runs lint/type-check/tests and returns a structured report. The orchestrator owns all retry and escalation logic.

#### Step 5a: Initial verification
```
Agent(
  model: "claude-haiku-4-5-20251001",
  skill: "mcd-verify",
  prompt: "Worktree: .worktrees/mcd-<slug>/"
)
```

Read the report. For each file with status `"failed"`:

#### Step 5b: Correction loop (per failed file)

**Attempts 1–2 — Haiku self-correction:**
```
Agent(
  model: "claude-haiku-4-5-20251001",
  skill: "mcd-fill",
  prompt: "Worktree: .worktrees/mcd-<slug>/\nFile: <file-path>\n\n<skeleton content>\n\nTypes:\n<relevant types>\n\nRelevant transitions:\n<transitions>\n\nCORRECTION ATTEMPT <N>:\nError output:\n<full error log from verify>\n\nPrevious attempt code:\n<full file content>\n\nFix the errors while strictly following the CONTRACT pseudocode. Do not change function signatures."
)
```
After each correction, re-run mcd-verify scoped to that file only:
```
Agent(
  model: "claude-haiku-4-5-20251001",
  skill: "mcd-verify",
  prompt: "Worktree: .worktrees/mcd-<slug>/\nFile: <file-path>"
)
```
If it passes → continue to next failed file.

**Attempt 3 — Pause and surface to user:**

Do NOT attempt another self-correction. Show:
```
=== MCD-Flow: Verification Failed ===
File: <file-path>
Attempts: 3

Error:
<full error log>

Diff (attempt 1 → attempt 3):
<diff>
```

Ask: **"Retry this file with Sonnet (claude-sonnet-4-6)? [y/n/skip]"**

- **y** → re-dispatch fill with `claude-sonnet-4-6`, re-verify
  - If Sonnet fails → ask: **"Escalate to Opus (claude-opus-4-6)? [y/n/skip]"**
    - **y** → re-dispatch with `claude-opus-4-6`, re-verify
    - **n / skip** → mark `"skipped"` in registry, continue
- **n** → ask: **"Skip this file or abort the pipeline? [skip/abort]"**
  - **skip** → mark `"skipped"`, continue
  - **abort** → stop pipeline, print summary
- **skip** → mark `"skipped"`, continue

---

## Step 3: Completion Gate

When Phase 5 exits cleanly, print:
```
=== MCD-Flow Pipeline Complete ===
Branch:   mcd/<slug>
Worktree: .worktrees/mcd-<slug>/

All tests pass. Merge mcd/<slug> → main? [y/n]
```

- **y** →
  ```bash
  git checkout main
  git merge mcd/<slug> --no-ff -m "feat: <feature name> (MCD-Flow)"
  git worktree remove .worktrees/mcd-<slug>
  git branch -d mcd/<slug>
  ```
  Print: `Merged and cleaned up.`

- **n** →
  ```
  Branch preserved: mcd/<slug>
  Worktree preserved: .worktrees/mcd-<slug>/
  To open a PR: gh pr create --head mcd/<slug>
  ```

---

## Individual Phase Re-runs

When invoked directly as `/mcd-discover`, `/mcd-plan`, `/mcd-skeleton`, `/mcd-audit`, `/mcd-fill`, or `/mcd-verify`:

1. Verify this is a git repository (`git rev-parse --is-inside-work-tree`). If not → print error and abort.
2. Check `.worktrees/` for any `mcd-*` directory.
3. If found → run the phase inside that worktree.
4. If none found → create a new worktree: `mcd/adhoc-<timestamp>`.
5. Dispatch the appropriate sub-agent with model and worktree path.
