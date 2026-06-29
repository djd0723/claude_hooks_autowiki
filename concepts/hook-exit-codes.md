---
type: concept
title: "Hook Exit Codes and Output Format"
created: 2026-06-29
updated: 2026-06-29
tags: [hooks, exit-codes, json-output, decisions]
source_count: 1
sources:
  - sources/clean/code-claude-com-docs-en-hooks-guide-md.md
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
| **0** | No objection; action proceeds normally. For `PreToolUse` this does NOT approve the call — the normal permission flow still applies. For `UserPromptSubmit`, `UserPromptExpansion`, and `SessionStart`, stdout is added to Claude's context. |
| **2** | Action is blocked. Write a reason to stderr; Claude receives it as feedback so it can adjust. Some events cannot be blocked: `SessionStart`, `Setup`, `Notification`, and others — exit 2 shows stderr to the user and execution continues. |
| **other** | Action proceeds. The transcript shows a `<hook name> hook error` notice followed by the first line of stderr; full stderr goes to the debug log. |

## Structured JSON output (exit 0 + stdout)

For more control than exit codes alone, exit 0 and write a JSON object to stdout. Claude Code ignores JSON when you exit 2 — don't mix them.

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

Use a top-level `decision: "block"` field.

### `UserPromptSubmit` — inject context

Use `additionalContext` to inject text into Claude's context rather than making a permission decision.

## Key constraints

- `"allow"` in a hook does not override deny rules from settings. Deny rules from any scope (including managed settings) always take precedence over hook approvals.
- `PostToolUse` hooks cannot undo actions — the tool has already executed.
- When multiple `PreToolUse` hooks return `updatedInput` to rewrite tool arguments, the last one to finish wins (non-deterministic due to parallel execution).

## Debug

- Toggle `Ctrl+O` in the transcript view for per-hook success/error summaries.
- Run `claude --debug-file /tmp/claude.log` for full execution details (matched hooks, exit codes, stdout, stderr).

## Related concepts

- [Hook Lifecycle Events](./hook-lifecycle-events.md) — which events exist
- [Hook Types](./hook-types.md) — command vs. prompt vs. agent vs. http
- [Hook Matchers](./hook-matchers.md) — filtering when hooks fire
- [Hook Scope](./hook-scope.md) — configuration location
