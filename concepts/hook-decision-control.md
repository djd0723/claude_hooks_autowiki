---
type: concept
title: "Hook Decision Control"
created: 2026-06-29
updated: 2026-06-29
tags: [hooks, decisions, block, allow, deny, permission, control]
source_count: 1
sources:
  - sources/clean/code-claude-com-docs-en-hooks-md.md
---

# Hook Decision Control

Hooks can do more than log — they can allow, block, modify, or redirect Claude Code's behavior. Different events use different decision patterns. This page is a quick reference for which pattern each event uses.

## Decision patterns at a glance

| Events | Decision pattern | Key fields |
| :----- | :--------------- | :--------- |
| `UserPromptSubmit`, `UserPromptExpansion`, `PostToolUse`, `PostToolUseFailure`, `PostToolBatch`, `Stop`, `SubagentStop`, `ConfigChange`, `PreCompact` | Top-level `decision` | `decision: "block"`, `reason` |
| `TeammateIdle`, `TaskCreated`, `TaskCompleted` | Exit code or `continue: false` | Exit code 2 blocks with stderr feedback; `{"continue": false, "stopReason": "..."}` stops entirely |
| `PreToolUse` | `hookSpecificOutput` | `permissionDecision` (allow/deny/ask/defer), `permissionDecisionReason`, `updatedInput` |
| `PermissionRequest` | `hookSpecificOutput` | `decision.behavior` (allow/deny), `updatedInput`, `updatedPermissions`, `message`, `interrupt` |
| `PermissionDenied` | `hookSpecificOutput` | `retry: true` tells the model it may retry |
| `WorktreeCreate` | Path return | Print path on stdout (command) or return `hookSpecificOutput.worktreePath` (HTTP) |
| `Elicitation` | `hookSpecificOutput` | `action` (accept/decline/cancel), `content` (form field values for accept) |
| `ElicitationResult` | `hookSpecificOutput` | `action`, `content` (override the user's response) |
| `MessageDisplay` | `hookSpecificOutput` | `displayContent` replaces on-screen text only |
| `SessionStart`, `Setup`, `SubagentStart` | Context only | `additionalContext` injects context; no blocking |
| `WorktreeRemove`, `Notification`, `SessionEnd`, `PostCompact`, `InstructionsLoaded`, `StopFailure`, `CwdChanged`, `FileChanged` | None | No decision control; side effects only |

## `PreToolUse` — the four outcomes

`PreToolUse` is the richest decision event. The `hookSpecificOutput.permissionDecision` field has four values:

| Value | Effect |
| :---- | :----- |
| `"allow"` | Skip the permission prompt. Deny and ask rules from settings still apply |
| `"deny"` | Prevent the tool call. Reason shown to Claude |
| `"ask"` | Show the permission prompt to the user. Permission prompt includes a `[User]`/`[Project]`/`[Plugin]` source label |
| `"defer"` | Non-interactive mode (`-p`) only: exits the process with tool call preserved for the Agent SDK caller to resume. In interactive mode, logs a warning and ignores the value |

When multiple hooks return different decisions, the most restrictive wins: `deny > defer > ask > allow`.

> Note: `PreToolUse` previously used top-level `decision` and `reason` fields. These are deprecated — use `hookSpecificOutput.permissionDecision` and `hookSpecificOutput.permissionDecisionReason` instead.

### Defer round-trip (Agent SDK)

`"defer"` is for Agent SDK integrations that need to collect user input through their own UI:

1. Claude calls a tool (commonly `AskUserQuestion`). The `PreToolUse` hook fires.
2. Hook returns `permissionDecision: "defer"`. Process exits with `stop_reason: "tool_deferred"`.
3. Caller reads `deferred_tool_use` from the SDK result, surfaces the question, collects the answer.
4. Caller runs `claude -p --resume <session-id>`. Same tool fires `PreToolUse` again.
5. Hook returns `permissionDecision: "allow"` with the answer in `updatedInput`. Tool executes.

`"defer"` only works when Claude makes a single tool call in the turn — batched calls cannot be individually deferred.

## `PermissionRequest` — auto-approving dialogs

`PermissionRequest` fires when a permission dialog is about to appear. Use it to approve or deny on the user's behalf without showing the dialog.

The `updatedPermissions` field accepts entries that persist the decision:

| `type` | Effect |
| :----- | :----- |
| `addRules` | Adds allow/deny/ask rules |
| `replaceRules` | Replaces all rules of a given behavior |
| `removeRules` | Removes matching rules |
| `setMode` | Changes the permission mode for the session |
| `addDirectories` | Adds working directories |
| `removeDirectories` | Removes working directories |

Each entry has a `destination`: `session` (in-memory), `localSettings`, `projectSettings`, or `userSettings`.

## Content rewriting

Some events can rewrite content rather than just block or allow:

| Event | Field | Rewrites |
| :---- | :---- | :------- |
| `PreToolUse` | `updatedInput` in `hookSpecificOutput` | Tool arguments before execution |
| `PermissionRequest` | `decision.updatedInput` | Tool arguments before execution |
| `PostToolUse` | `updatedToolOutput` in `hookSpecificOutput` | Tool result before Claude sees it |
| `MessageDisplay` | `displayContent` in `hookSpecificOutput` | On-screen display only; transcript unchanged |

> For redaction: intercept at `PreToolUse` for outbound inputs, `PostToolUse` for inbound results.

## Prompt and agent hook decisions

Prompt (`type: "prompt"`) and agent (`type: "agent"`) hooks return `{"ok": true}` to allow or `{"ok": false, "reason": "..."}` to block. The `ok: false` effect depends on the event:

| Event | `ok: false` effect |
| :---- | :----------------- |
| `Stop`, `SubagentStop` | Reason fed back to Claude as next instruction; turn continues |
| `PreToolUse` | Tool call denied; reason returned as tool error |
| `PostToolUse` | Turn ends; reason appears as warning. Set `continueOnBlock: true` to feed back instead |
| `PostToolBatch`, `UserPromptSubmit`, `UserPromptExpansion` | Turn ends; reason appears as warning |
| `PostToolUseFailure`, `TaskCreated`, `TaskCompleted` | Reason returned as tool error |
| `TeammateIdle` | Teammate stops by default. Set `continueOnBlock: true` to keep working |
| `PermissionRequest` | `ok: false` has no effect — use a command hook for deny |
| `PermissionDenied` | Output discarded — use a command hook returning `hookSpecificOutput.retry: true` |

## `Stop` — continuing vs. stopping

Two ways to keep the conversation going from a `Stop` hook:

- `decision: "block"` + `reason`: feeds `reason` to Claude as a hook error. The transcript shows a hook error notice.
- `hookSpecificOutput.additionalContext`: feeds text to Claude as `Stop hook feedback`. The transcript labels it as feedback, no error notice shown. Use this for expected post-turn actions like "run the test suite before finishing".

Both paths use the same 8-consecutive-block cap and `stop_hook_active` guard.

## Related concepts

- [Hook Exit Codes](./hook-exit-codes.md) — exit code meanings and per-event blocking table
- [Hook Input and Output](./hook-input-output.md) — full JSON input/output schema
- [Hook Lifecycle Events](./hook-lifecycle-events.md) — the full event list
- [Hook Types](./hook-types.md) — command vs. prompt vs. agent
