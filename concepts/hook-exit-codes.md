---
type: concept
title: "Hook Exit Codes and Output Format"
created: 2026-06-29
updated: 2026-06-29
tags: [hooks, exit-codes, json-output, decisions]
source_count: 2
sources:
  - sources/clean/code-claude-com-docs-en-hooks-guide-md.md
  - sources/clean/code-claude-com-docs-en-hooks-md.md
---

# Hook Exit Codes and Output Format

Command hooks communicate with Claude Code through stdin, stdout, stderr, and exit codes.

## Input

Claude Code passes event-specific JSON to the hook's stdin. Every event includes common fields:

```json
{
  "session_id": "abc123",
  "cwd": "/Users/sarah/myproject",
  "hook_event_name": "PreToolUse",
  "tool_name": "Bash",
  "tool_input": {
    "command": "npm test"
  }
}
```

`UserPromptSubmit` hooks get `prompt` text; `SessionStart` hooks get `source` (startup, resume, clear, compact). See the Hooks reference for full per-event schemas.

## Exit codes

| Exit code | Meaning |
| :-------- | :------ |
| **0** | Success. Claude Code parses stdout for JSON output fields. For `UserPromptSubmit`, `UserPromptExpansion`, and `SessionStart`, stdout is added as context Claude can see |
| **2** | Blocking error. Stderr is fed back to Claude as an error message. The exact effect depends on the event (see table below). Only exit code 2 blocks; exit code 1 is a non-blocking error |
| **other** | Non-blocking error for most events. The transcript shows a `<hook name> hook error` notice followed by the first line of stderr; full stderr goes to the debug log |

> Note: For `WorktreeCreate`, **any** non-zero exit code aborts worktree creation.

### Exit code 2 behavior per event

| Hook event | Can block? | What happens on exit 2 |
| :--------- | :--------- | :--------------------- |
| `PreToolUse` | Yes | Blocks the tool call |
| `PermissionRequest` | Yes | Denies the permission |
| `UserPromptSubmit` | Yes | Blocks prompt processing and erases the prompt |
| `UserPromptExpansion` | Yes | Blocks the expansion |
| `Stop` | Yes | Prevents Claude from stopping, continues the conversation |
| `SubagentStop` | Yes | Prevents the subagent from stopping |
| `TeammateIdle` | Yes | Prevents the teammate from going idle |
| `TaskCreated` | Yes | Rolls back the task creation |
| `TaskCompleted` | Yes | Prevents the task from being marked completed |
| `ConfigChange` | Yes | Blocks the configuration change (except `policy_settings`) |
| `PostToolBatch` | Yes | Stops the agentic loop before the next model call |
| `PreCompact` | Yes | Blocks compaction |
| `Elicitation` | Yes | Denies the elicitation |
| `ElicitationResult` | Yes | Blocks the response (action becomes decline) |
| `WorktreeCreate` | Yes | Any non-zero exit code fails creation |
| `PostToolUse` | No | Shows stderr to Claude (tool already ran) |
| `PostToolUseFailure` | No | Shows stderr to Claude |
| `StopFailure` | No | Output and exit code are ignored |
| `PermissionDenied` | No | Exit code ignored; use JSON `hookSpecificOutput.retry: true` for retry |
| `SessionStart`, `Setup`, `SessionEnd` | No | Shows stderr to user only |
| `Notification`, `SubagentStart` | No | Shows stderr to user only |
| `CwdChanged`, `FileChanged` | No | Shows stderr to user only |
| `PostCompact`, `InstructionsLoaded`, `MessageDisplay` | No | Shows stderr to user only |

## HTTP response handling

HTTP hooks use HTTP status codes and response bodies instead of exit codes and stdout:

| Response | Equivalent to |
| :------- | :------------ |
| 2xx with empty body | exit 0 with no output |
| 2xx with plain text body | exit 0, text added as context |
| 2xx with JSON body | exit 0, parsed using the same JSON output schema |
| Non-2xx, connection failure, timeout | Non-blocking error, execution continues |

Unlike command hooks, HTTP hooks cannot signal a blocking error via status codes alone — return a 2xx with `decision: "block"` or `hookSpecificOutput.permissionDecision: "deny"` in the body.

## Structured JSON output (exit 0 + stdout)

For more control than exit codes alone, exit 0 and write a JSON object to stdout. Claude Code ignores JSON when you exit 2 — don't mix them.

### Universal JSON output fields

| Field | Default | Description |
| :---- | :------ | :---------- |
| `continue` | `true` | If `false`, Claude stops processing entirely after the hook runs, regardless of event-specific decisions |
| `stopReason` | none | Message shown to the user when `continue` is `false`. Not shown to Claude |
| `suppressOutput` | `false` | If `true`, hides the hook's stdout from the transcript (still appears in debug log) |
| `systemMessage` | none | Warning message shown to the user |
| `terminalSequence` | none | Terminal escape sequence for Claude Code to emit (desktop notification, window title, bell). Restricted to OSC `0`/`1`/`2`/`9`/`99`/`777` and BEL. Hooks cannot write to `/dev/tty` directly — use this instead |

### `PreToolUse` — permission decisions

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "deny",
    "permissionDecisionReason": "Use rg instead of grep for better performance"
  }
}
```

`permissionDecision` values for `PreToolUse`:
- `"allow"` — skip the interactive permission prompt (deny/ask rules from settings still apply)
- `"deny"` — cancel the tool call, send reason to Claude
- `"ask"` — show the permission prompt to the user
- `"defer"` — non-interactive mode only: exits the process with tool call preserved for Agent SDK wrapper

### `PermissionRequest` — auto-approve permission dialogs

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PermissionRequest",
    "decision": {
      "behavior": "allow"
    }
  }
}
```

Can also include `updatedPermissions` to set the session's permission mode:

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PermissionRequest",
    "decision": {
      "behavior": "allow",
      "updatedPermissions": [
        { "type": "setMode", "mode": "acceptEdits", "destination": "session" }
      ]
    }
  }
}
```

### `PostToolUse` / `Stop` — block after the fact

Use a top-level `decision: "block"` field. `PostToolUse` also supports `updatedToolOutput` to replace what Claude sees:

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PostToolUse",
    "updatedToolOutput": {
      "stdout": "[redacted]",
      "stderr": "",
      "interrupted": false,
      "isImage": false
    }
  }
}
```

The replacement value must match the tool's output shape. The tool has already run — `updatedToolOutput` only changes what Claude sees.

### `UserPromptSubmit` — inject context

Use `additionalContext` to inject text into Claude's context rather than making a permission decision.

### `PreToolUse` — modify tool input

The `updatedInput` field inside `hookSpecificOutput` replaces the tool's arguments before execution. Combine with `permissionDecision: "allow"` to auto-approve with a modified input:

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "allow",
    "updatedInput": {
      "command": "npm run lint --fix"
    }
  }
}
```

## Key constraints

- `"allow"` in a hook does not override deny rules from settings. Deny rules from any scope (including managed settings) always take precedence over hook approvals.
- `PostToolUse` hooks cannot undo actions — the tool has already executed.
- When multiple `PreToolUse` hooks return `updatedInput` to rewrite tool arguments, the last one to finish wins (non-deterministic due to parallel execution).
- `Stop` hooks have a block cap of **8 consecutive blocks**. After 8 consecutive blocks, Claude Code stops calling the `Stop` hook for the session. Check the `stop_hook_active` field in the hook input to detect when a block cascade is in progress and avoid triggering additional blocks.

## Debug

- Toggle `Ctrl+O` in the transcript view for per-hook success/error summaries.
- Run `claude --debug-file /tmp/claude.log` for full execution details (matched hooks, exit codes, stdout, stderr).

## Related concepts

- [Hook Lifecycle Events](./hook-lifecycle-events.md) — which events exist
- [Hook Types](./hook-types.md) — command vs. prompt vs. agent vs. http
- [Hook Matchers](./hook-matchers.md) — filtering when hooks fire
- [Hook Scope](./hook-scope.md) — configuration location
- [Hook Input and Output](./hook-input-output.md) — full JSON input/output schema
- [Hook Decision Control](./hook-decision-control.md) — per-event decision patterns
