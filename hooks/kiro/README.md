# Kiro IDE / CLI Hooks

> Part of [`hooks/`](../README.md) — see also [`src/hooks/`](../../src/hooks/README.md) for installation code

## Specifics

- Uses the `rtk hook kiro` Rust binary (not a shell script) — no `jq` dependency
- Dual mechanism: **steering file** (`.kiro/steering/rtk.md`, prompt-level guidance) as primary integration, plus an optional **PreToolUse hook** (`.kiro/hooks/rtk-rewrite.kiro.hook`) for ask-with-suggestion reinforcement
- Hook returns `permissionDecision: "ask"` with the suggested `rtk` command (Kiro's PreToolUse API does not support transparent command rewrite / `updatedInput`)
- Exits silently (exit 0, no output) on any failure: invalid JSON, missing command, no rewrite match, stdin > 1 MiB
- Structured for future transparent rewrite if Kiro exposes an `updatedInput`-style field

## Mechanism

The Kiro PreToolUse hook is registered via a JSON config file at `.kiro/hooks/rtk-rewrite.kiro.hook`. When Kiro's shell execution tool is invoked, the hook triggers `rtk hook kiro`, which:

1. Reads the JSON payload from stdin (capped at 1 MiB)
2. Extracts the shell command from `tool_input.command`
3. Delegates to the shared rewrite decision flow (`decide_hook_action`)
4. Emits an ask-with-suggestion response if a rewrite exists, or produces no output (exit 0) otherwise

## JSON Formats

### Input (stdin — Kiro → hook)

Kiro sends the session context, hook event, tool name, and tool input:

```json
{
  "session_id": "0f2c…",
  "hook_event_name": "PreToolUse",
  "tool_name": "executeBash",
  "tool_input": { "command": "git status" }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `session_id` | string | Kiro session identifier (ignored by hook) |
| `hook_event_name` | string | Always `"PreToolUse"` for this hook |
| `tool_name` | string | Name of the tool being invoked (matched by `executeBash`) |
| `tool_input` | object | Tool arguments; `command` is the shell command string |

### Output (stdout — hook → Kiro) — ask-with-suggestion

When the command has an RTK equivalent:

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "ask",
    "permissionDecisionReason": "RTK: considere usar `rtk git status` para economizar 60-90% de tokens"
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `hookSpecificOutput.hookEventName` | string | Echo of the hook event (`"PreToolUse"`) |
| `hookSpecificOutput.permissionDecision` | string | `"ask"` — prompts the user before executing |
| `hookSpecificOutput.permissionDecisionReason` | string | Human-readable suggestion with the `rtk` command |

### Output — no rewrite

When no rewrite applies (command has no RTK equivalent, is already prefixed with `rtk`, contains unattestable constructs, heredoc, or on any error): **empty stdout** and exit code 0. The original command executes unmodified.

## Exit Code Contract

`rtk hook kiro` exits with code 0 in **all** paths — including errors, invalid input, and no-match cases. The hook never blocks command execution.

| Condition | Behavior |
|-----------|----------|
| Valid command with RTK equivalent | stdout: ask JSON, exit 0 |
| No RTK equivalent | no output, exit 0 |
| Command already prefixed with `rtk` | no output, exit 0 |
| Unattestable construct / heredoc | no output, exit 0 |
| Invalid JSON input | no output, exit 0 |
| Empty stdin | no output, exit 0 |
| Stdin exceeds 1 MiB | no output, exit 0 |
| `tool_name` is not a shell tool | no output, exit 0 |
