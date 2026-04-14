# Sub-Agent Dispatch (how phase invocation actually works)

Phases of MCD-Flow run as sub-agents dispatched via the Claude Code `Agent` tool. This doc specifies the exact dispatch shape — readers of the orchestrator command should treat this as the contract.

## The Agent tool

The `Agent` tool accepts:

| Param | Required | Value for MCD-Flow |
|-------|----------|--------------------|
| `description` | yes | `"MCD Phase <N>: <name>"` |
| `subagent_type` | yes | `"general-purpose"` |
| `model` | optional | `"opus"`, `"sonnet"`, or `"haiku"` — short alias only |
| `prompt` | yes | phase instructions + context, fully inlined |

There is no `skill:` parameter. Sub-agents do not inherit slash-command access, so "invoke `/mcd-discover` as a sub-agent" is not a valid pattern. Instead, the orchestrator reads the corresponding phase command file and inlines its body as the sub-agent prompt.

## Dispatch recipe (orchestrator side)

For each phase:

1. **Locate the phase instructions.** The plugin install path is exposed as `${CLAUDE_PLUGIN_ROOT}`. Read `${CLAUDE_PLUGIN_ROOT}/commands/<phase>.md`. Strip the YAML frontmatter (everything between the leading `---` and the second `---`). The remaining markdown body is the phase brief.

2. **Compose the prompt** as:
   ```
   <phase brief body>

   ---

   ## Execution context

   Worktree: <absolute path to worktree>
   Feature: <feature slug>
   Mode: <greenfield|diff>
   Config: <absolute path to .mcd/config.json, or "not-yet-written" for Phase 1>

   <any phase-specific extras, e.g. for mcd-fill: the skeleton file content, the types, the relevant transitions>
   ```

3. **Dispatch:**
   ```
   Agent(
     description: "MCD Phase 2: Skeleton",
     subagent_type: "general-purpose",
     model: "sonnet",
     prompt: <composed prompt from step 2>
   )
   ```

4. **Wait** for the sub-agent to return. Read its final message + the registry/files it wrote into the worktree. Proceed to the next phase only after the gate (if any) is cleared.

## Model routing

| Phase | Model alias | Full ID (for docs only) |
|-------|-------------|-------------------------|
| 0 Discover | `opus` | claude-opus-4-6 |
| 1 Plan | `opus` | claude-opus-4-6 |
| 2 Skeleton | `sonnet` | claude-sonnet-4-6 |
| 3 Audit | `opus` | claude-opus-4-6 |
| 4 Fill | `haiku` | claude-haiku-4-5-20251001 |
| 5 Verify | `haiku` | claude-haiku-4-5-20251001 |

Always pass the **short alias** to the `Agent` tool's `model` parameter.

## Parallel fill (Phase 4)

In a single orchestrator turn, emit multiple `Agent(...)` tool calls — one per file at the current DAG level. The Claude Code harness executes tool calls in parallel when issued in the same turn. Wait for all to return before dispatching the next level.

Each fill sub-agent writes to its own registry shard (`.mcd/registry/<slugified-file-path>.json`). After all level N fills complete, the orchestrator merges shards into `.mcd/registry.json` to avoid last-write-wins corruption.

## Fallback when `${CLAUDE_PLUGIN_ROOT}` is unset

If the env var is empty, locate the plugin by walking up from the current file until finding a `.claude-plugin/plugin.json`. If still not found, error out with instructions to the user.
