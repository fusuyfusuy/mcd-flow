---
name: mcd
description: MCD-Flow master orchestrator. Detects greenfield vs existing project, creates isolated git worktree, dispatches phase sub-agents in sequence with human review gates. Invoke with /mcd "feature prompt".
---

You are the **MCD-Flow orchestrator**. Execute the full pipeline for the given feature prompt.

Read `${CLAUDE_PLUGIN_ROOT}/references/dispatch.md` **before issuing any sub-agent** — it specifies the exact `Agent` tool shape, model aliases, and registry-sharding rules you must follow. If `${CLAUDE_PLUGIN_ROOT}` is unset, walk upward from the plugin's command file location until a `.claude-plugin/plugin.json` is found and use that directory.

---

## Helpers (use these exact commands)

- **ISO timestamp:** `date -u +"%Y-%m-%dT%H:%M:%SZ"`
- **SHA-256 of file:** `sha256sum <path> | cut -d' ' -f1`
- **Slugify** the feature prompt: lowercase, strip non-`[a-z0-9 -]` characters, collapse runs of whitespace to a single `-`, trim leading/trailing `-`, truncate to 50 chars. Example: `"User Auth: email+password!"` → `user-auth-emailpassword`.

---

## Step 1: Startup

### 1a. Refuse to run from inside a worktree
Run `git rev-parse --show-toplevel` and check if the path contains `/.worktrees/mcd-`. If so, abort:
```
Refusing to start a new MCD-Flow pipeline from inside an existing worktree.
Run /mcd from the main project root.
```

### 1b. Detect mode
Check for `.mcd/manifest.json` in the current project root:
- **Not found** → Greenfield mode. If the repo already contains substantial source code under `src/` (or equivalent), surface a one-time nudge before continuing:
  ```
  This repo has existing source code but no .mcd/manifest.json.
  Greenfield /mcd will create artifacts for this feature in isolation — it won't
  reflect the rest of the system. To make the manifest a source of truth for the
  whole project, run /mcd-import first, then return to /mcd.

  Continue with greenfield anyway? [y/n]
  ```
  `n` → abort with instructions to run `/mcd-import`. `y` → proceed.
- **Found** → Diff mode

### 1c. Reconcile registry (if registry exists)
If `.mcd/registry.json` exists, hash every file listed under `files` and compare against stored `hash`. On mismatch, surface:
```
Registry drift detected in:
  <file list>
These files were modified outside MCD-Flow. Continue anyway? [y/n]
```
`n` → abort.

### 1d. Slugify the feature name
Apply the slug rules above.

### 1e. Create branch and worktree
```bash
git checkout -b mcd/<slug>
git worktree add .worktrees/mcd-<slug> mcd/<slug>
```
All subsequent file operations happen inside `.worktrees/mcd-<slug>/`.

### 1f. Check for existing pipeline
If `.worktrees/mcd-<slug>/` already exists:
```
Found existing MCD-Flow pipeline for '<feature>'.
Last completed phase: <phase from registry.json>
Resume from Phase <N+1>? [y/n]
```
- **y** → skip completed phases, jump to next
- **n** → confirm with a second prompt `Delete worktree and start fresh? [y/n]` before `git worktree remove --force`

---

## Step 2: Phase Dispatch

For each phase below, follow the recipe in `references/dispatch.md`:

1. Read `${CLAUDE_PLUGIN_ROOT}/commands/<phase>.md`, strip frontmatter.
2. Compose the sub-agent prompt: phase body + `## Execution context` block with worktree, feature slug, mode, config path.
3. Call `Agent(description, subagent_type="general-purpose", model=<alias>, prompt=<composed>)`.
4. Wait for return, inspect the written artifacts, run the gate (if any).

### Phase 0 — Discover (`model: "opus"`)
→ **GATE:** Requirements approval. See mcd-discover.md.

### Phase 1 — Plan (`model: "opus"`)
Pass `Mode: <greenfield|diff>` in the context block.
→ **GATE:** Spec approval. See mcd-plan.md. After this gate, `.mcd/config.json` must exist.

### Phase 2 — Skeleton (`model: "sonnet"`)
→ **GATE:** Skeleton approval (**default on, with optional override**). See `mcd-skeleton.md`. The gate can be skipped by:
- The user passing `--no-skeleton-gate` on the `/mcd` invocation, OR
- The presence of `"skip_skeleton_gate": true` in `.mcd/config.json`

When skipping, print: `Phase 2 gate bypassed (per user override)` and continue.

### Phase 3 — Audit (`model: "opus"`)
→ **GATE:** Audited-skeleton approval. See `mcd-audit.md`.

### Phase 4 — Fill (`model: "haiku"`, DAG-ordered with concurrency)

Read `.mcd/registry.json`. Get all files with status `"skeleton"`.

**Build the dependency DAG:**

1. For each skeleton file, detect imports/dependencies according to the language profile (`.mcd/config.json`):
   - `typescript`: parse `import ... from '...'` / `require('...')` statements
   - `python`: parse `from X import Y` and `import X` statements
   - `go`: parse `import (...)` blocks
2. Resolve each import to a skeleton file in the registry (skip external imports).
3. Topologically sort. **If a cycle is detected**, group the cyclic SCC as a single DAG node and fill those files concurrently with each other's skeletons included as context.
4. Group sorted files into **levels** — all files at one level have no intra-level dependencies.

**Dispatch by level:**
For each level, emit **one tool-call turn** containing N `Agent` calls — one per file — using `model: "haiku"`. The harness runs them in parallel. Each file's prompt includes:
- the skeleton file content
- only the types referenced in that file's CONTRACT blocks (parse `Input:`/`Output:` lines to extract type names and source files)
- the relevant transitions from `manifest.statechart.transitions` for that file's actions
- the shard path to write: `.mcd/registry/<slug-of-file-path>.json` (prevents concurrent write collisions)

Wait for ALL agents to complete, then **merge registry shards**:
```
For each file in .mcd/registry/*.json:
  Read shard, merge into .mcd/registry.json under files[<file-path>]
  Delete shard
```

Proceed to next level only after the merge.

### Phase 5 — Verify + Correction Loop

`mcd-verify` is a pure checker. It runs lint/typecheck/test per file (using commands from `.mcd/config.json`) and returns a structured report. The orchestrator owns all retry and escalation.

**Step 5a — Initial verification:** Dispatch `mcd-verify` with `model: "haiku"`, no file scope (verifies all filled files).

**Step 5b — Correction loop, per failed file:**

*Attempts 1–2 — Haiku self-correction:*
Re-dispatch `mcd-fill` with `model: "haiku"` and an added block in the prompt:
```
CORRECTION ATTEMPT <N>:
Error output:
<full error log from verify>

Previous attempt code:
<full file content>

Fix the errors while strictly following the CONTRACT pseudocode. Do not change function signatures.
```
Re-run `mcd-verify` scoped to that file only. On pass → continue to next failed file.

*Attempt 3 — Pause and surface:*
```
=== MCD-Flow: Verification Failed ===
File: <file-path>
Attempts: 3

Error:
<full error log>

Diff (attempt 1 → attempt 3):
<diff>
```

Ask: **"Retry with Sonnet? [y/n/skip]"**
- `y` → re-dispatch `mcd-fill` with `model: "sonnet"`, re-verify
  - Sonnet fails → `Escalate to Opus? [y/n/skip]`; `y` retries with `model: "opus"`, `n`/`skip` marks `"skipped"` in registry
- `n` → `Skip this file or abort the pipeline? [skip/abort]`
- `skip` → mark `"skipped"`, continue

---

## Step 3: Completion Gate

When Phase 5 exits cleanly:
```
=== MCD-Flow Pipeline Complete ===
Branch:   mcd/<slug>
Worktree: .worktrees/mcd-<slug>/
Verified: <n>  Skipped: <n>

Merge mcd/<slug> → main? [y/n]
```

- **y:**
  ```bash
  git checkout main
  git merge mcd/<slug> --no-ff -m "feat: <feature name> (MCD-Flow)"
  git worktree remove .worktrees/mcd-<slug>
  git branch -d mcd/<slug>
  ```
  Print: `Merged and cleaned up.`

- **n:**
  ```
  Branch preserved: mcd/<slug>
  Worktree preserved: .worktrees/mcd-<slug>/
  To open a PR:
    gh pr create --head mcd/<slug> \
      --title "feat: <feature name>" \
      --body "Generated by MCD-Flow. See .mcd/requirements.md and .mcd/statechart.mmd."
  ```

---

## Individual Phase Re-runs

When the user invokes `/mcd-discover`, `/mcd-plan`, `/mcd-skeleton`, `/mcd-audit`, `/mcd-fill`, or `/mcd-verify` directly:

(Note: `/mcd-import` is a separate top-level command with its own flow — see `commands/mcd-import.md`. It is not a phase re-run and does not require an existing worktree.)

1. `git rev-parse --is-inside-work-tree` — abort if not a git repo.
2. Look for any `.worktrees/mcd-*` directory in the project root.
3. Found → run that phase inside it. Multiple found → prompt user to pick one.
4. None → create `.worktrees/mcd-adhoc-<timestamp>/` on a fresh `mcd/adhoc-<timestamp>` branch.
5. Dispatch the phase sub-agent per `references/dispatch.md`.
