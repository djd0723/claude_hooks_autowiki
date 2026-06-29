---
type: concept
title: "Hook Input and Output Protocol"
created: 2026-06-29
updated: 2026-06-29
tags: [hooks, json, input, output, stdin, stdout, context]
source_count: 1
sources:
  - sources/clean/code-claude-com-docs-en-hooks-md.md
---

# Hook Input and Output Protocol

Every hook receives JSON on stdin (command hooks) or as the POST body (HTTP hooks). This page documents the common fields present in every event and the full set of output fields available in the hook's response.

## Common input fields

These fields are present on every hook event:

| Field | Description |
| :---- | :---------- |
| `session_id` | Current session identifier |
| `transcript_path` | Path to the conversation JSON file |
| `cwd` | Working directory when the hook is invoked |
| `hook_event_name` | Name of the event that fired |
| `permission_mode` | Active permission mode: `"default"`, `"plan"`, `"acceptEdits"`, `"auto"`, `"dontAsk"`, or `"bypassPermissions"`. Not present on all events |
| `effort` | Object with `level` field: `"low"`, `"medium"`, `"high"`, `"xhigh"`, or `"max"`. Present on tool-use events when the model supports the effort parameter |

When running inside a subagent or with `--agent`:

| Field | Description |
| :---- | :---------- |
| `agent_id` | Unique identifier for the subagent. Present only when the hook fires inside a subagent |
| `agent_type` | Agent name (e.g. `"Explore"` or `"security-reviewer"`). Present when using `--agent` or inside a subagent |

### Example `PreToolUse` input

```json
{
  "session_id": "abc123",
  "transcript_path": "/home/user/.claude/projects/.../transcript.jsonl",
  "cwd": "/home/user/my-project",
  "permission_mode": "default",
  "hook_event_name": "PreToolUse",
  "tool_name": "Bash",
  "tool_input": {
    "command": "npm test"
  }
}
```

## JSON output fields

On exit 0, Claude Code reads a JSON object from the hook's stdout. Hook output strings are capped at 10,000 characters; longer values are saved to a file and replaced with a preview and path.

### Universal fields (apply to all events)

| Field | Default | Description |
| :---- | :------ | :---------- |
| `continue` | `true` | If `false`, Claude stops processing entirely after the hook runs. Takes precedence over all event-specific decisions |
| `stopReason` | none | Shown to the user when `continue` is `false`. Not shown to Claude |
| `suppressOutput` | `false` | Hides the hook's stdout from the transcript; still appears in debug log |
| `systemMessage` | none | Warning message shown to the user |
| `terminalSequence` | none | A terminal escape sequence emitted by Claude Code on the hook's behalf. Restricted to OSC `0`/`1`/`2`/`9`/`99`/`777` and BEL. Use instead of writing to `/dev/tty`, which is unavailable to hooks |

### `additionalContext`

The `additionalContext` field (inside `hookSpecificOutput`) passes a string into Claude's context window. Claude receives it as a system reminder at the point the hook fired. It does not appear as a chat message.

Where it is injected depends on the event:

| Event | Where injected |
| :---- | :------------- |
| `SessionStart`, `Setup`, `SubagentStart` | Start of conversation, before first prompt |
| `UserPromptSubmit`, `UserPromptExpansion` | Alongside the submitted prompt |
| `PreToolUse`, `PostToolUse`, `PostToolUseFailure`, `PostToolBatch` | Next to the tool result |
| `Stop`, `SubagentStop` | At end of turn; conversation continues so Claude can act on it |

When multiple hooks return `additionalContext` for the same event, Claude receives all values.

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PostToolUse",
    "additionalContext": "This file is generated. Edit src/schema.ts and run `bun generate` instead."
  }
}
```

> Use `additionalContext` for factual project state: "The deployment target is production" or "This repo uses `bun test`". Phrasing as imperative system commands can trigger Claude's prompt-injection defenses.

## Per-event output fields

### `SessionStart`

| Field | Description |
| :---- | :---------- |
| `additionalContext` | Added before first prompt |
| `initialUserMessage` | Used as first user message in non-interactive mode |
| `sessionTitle` | Sets the session title. Only for `startup` or `resume` source |
| `watchPaths` | Array of absolute paths to watch for `FileChanged` events |
| `reloadSkills` | When `true`, re-scans skill directories after SessionStart hooks complete |

### `PreToolUse`

| Field | Description |
| :---- | :---------- |
| `permissionDecision` | `"allow"`, `"deny"`, `"ask"`, or `"defer"` |
| `permissionDecisionReason` | Shown to user (allow/ask) or to Claude (deny) |
| `updatedInput` | Replaces the tool's input arguments before execution |
| `additionalContext` | Added alongside the tool result |

Multiple `PreToolUse` hooks: precedence is `deny > defer > ask > allow`.

### `PermissionRequest`

| Field | Description |
| :---- | :---------- |
| `decision.behavior` | `"allow"` or `"deny"` |
| `decision.updatedInput` | For `"allow"`: modifies tool input before execution |
| `decision.updatedPermissions` | For `"allow"`: array of permission update entries (addRules, replaceRules, setMode, addDirectories, etc.) |
| `decision.message` | For `"deny"`: tells Claude why |
| `decision.interrupt` | For `"deny"`: if `true`, stops Claude |

### `PostToolUse`

| Field | Description |
| :---- | :---------- |
| `decision` | `"block"` adds `reason` next to the tool result |
| `reason` | Explanation shown to Claude when `decision` is `"block"` |
| `additionalContext` | Added alongside the tool result |
| `updatedToolOutput` | Replaces the tool's output with the provided value before it is sent to Claude |

### `UserPromptSubmit`

| Field | Description |
| :---- | :---------- |
| `decision` | `"block"` prevents prompt processing and erases it |
| `reason` | Shown to user when `decision` is `"block"` |
| `additionalContext` | Added alongside the submitted prompt |
| `sessionTitle` | Sets the session title based on prompt content |
| `suppressOriginalPrompt` | If `true` and `decision` is `"block"`, omits original prompt from block message |

### `Stop` / `SubagentStop`

| Field | Description |
| :---- | :---------- |
| `decision` | `"block"` prevents Claude from stopping; requires `reason` |
| `reason` | Required with `"block"`. Tells Claude why it should continue |
| `hookSpecificOutput.additionalContext` | Non-error feedback that keeps the conversation going without triggering a hook error notice |

### `MessageDisplay`

| Field | Description |
| :---- | :---------- |
| `hookSpecificOutput.displayContent` | Replaces the displayed text on screen. Display-only: transcript and what Claude sees keep the original |

### `WorktreeCreate`

The hook must return the absolute path to the created worktree directory. Command hooks print it on stdout; HTTP hooks return `hookSpecificOutput.worktreePath`.

### `CwdChanged` / `FileChanged`

| Field | Description |
| :---- | :---------- |
| `watchPaths` | Array of absolute paths. Replaces the current dynamic watch list |

## Related concepts

- [Hook Exit Codes](./hook-exit-codes.md) — exit code meanings and per-event blocking
- [Hook Lifecycle Events](./hook-lifecycle-events.md) — the full event list
- [Hook Decision Control](./hook-decision-control.md) — per-event decision patterns
- [Hook Types](./hook-types.md) — command vs. prompt vs. agent vs. http
- [Hook Matchers](./hook-matchers.md) — filtering which hooks fire
- [Hook Scope](./hook-scope.md) — configuration location and permission interaction
- [SDK Callback Hooks](./sdk-callback-hooks.md) — SDK callbacks return this same JSON output format
