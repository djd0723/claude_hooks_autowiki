---
type: concept
title: "Hook Lifecycle Events"
created: 2026-06-29
updated: 2026-06-29
tags: [hooks, lifecycle, events, automation]
source_count: 1
sources:
  - sources/clean/code-claude-com-docs-en-hooks-guide-md.md
---

# Hook Lifecycle Events

Claude Code fires hooks at specific lifecycle points. Each event passes JSON data to the hook's stdin and reads the hook's output to determine what to do next.

## The event table

| Event | When it fires |
| :---- | :------------ |
| `SessionStart` | When a session begins or resumes |
| `Setup` | When you start Claude Code with `--init-only`, or with `--init` or `--maintenance` in `-p` mode |
| `UserPromptSubmit` | When you submit a prompt, before Claude processes it |
| `UserPromptExpansion` | When a user-typed command expands into a prompt, before it reaches Claude. Can block the expansion |
| `PreToolUse` | Before a tool call executes. Can block it |
| `PermissionRequest` | When a permission dialog appears |
| `PermissionDenied` | When a tool call is denied by the auto mode classifier. Return `{retry: true}` to tell the model it may retry |
| `PostToolUse` | After a tool call succeeds |
| `PostToolUseFailure` | After a tool call fails |
| `PostToolBatch` | After a full batch of parallel tool calls resolves, before the next model call |
| `Notification` | When Claude Code sends a notification |
| `MessageDisplay` | While assistant message text is displayed |
| `SubagentStart` | When a subagent is spawned |
| `SubagentStop` | When a subagent finishes |
| `TaskCreated` | When a task is being created via `TaskCreate` |
| `TaskCompleted` | When a task is being marked as completed |
| `Stop` | When Claude finishes responding |
| `StopFailure` | When the turn ends due to an API error. Output and exit code are ignored |
| `TeammateIdle` | When an agent team teammate is about to go idle |
| `InstructionsLoaded` | When a CLAUDE.md or `.claude/rules/*.md` file is loaded into context |
| `ConfigChange` | When a configuration file changes during a session |
| `CwdChanged` | When the working directory changes |
| `FileChanged` | When a watched file changes on disk. The `matcher` field specifies which filenames to watch |
| `WorktreeCreate` | When a worktree is being created via `--worktree` or `isolation: "worktree"`. Replaces default git behavior |
| `WorktreeRemove` | When a worktree is being removed, either at session exit or when a subagent finishes |
| `PreCompact` | Before context compaction |
| `PostCompact` | After context compaction completes |
| `Elicitation` | When an MCP server requests user input during a tool call |
| `ElicitationResult` | After a user responds to an MCP elicitation, before the response is sent back to the server |
| `SessionEnd` | When a session terminates |

## Common hook event categories

**Session lifecycle**: `SessionStart`, `Setup`, `SessionEnd`  
**Tool control**: `PreToolUse`, `PostToolUse`, `PostToolUseFailure`, `PostToolBatch`  
**Permission control**: `PermissionRequest`, `PermissionDenied`  
**User interaction**: `UserPromptSubmit`, `UserPromptExpansion`, `Notification`, `MessageDisplay`  
**Agent/task**: `SubagentStart`, `SubagentStop`, `TaskCreated`, `TaskCompleted`, `Stop`, `StopFailure`  
**Environment**: `CwdChanged`, `FileChanged`, `ConfigChange`, `InstructionsLoaded`  
**Compaction**: `PreCompact`, `PostCompact`  
**Worktrees**: `WorktreeCreate`, `WorktreeRemove`  
**MCP elicitation**: `Elicitation`, `ElicitationResult`

## Key behaviors

- Multiple matching hooks for the same event run **in parallel**, not sequentially
- Identical hook commands are automatically deduplicated
- For `PreToolUse` permission decisions, the most restrictive answer wins: `deny > defer > ask > allow`
- `PermissionRequest` hooks do not fire in non-interactive mode (`-p`); use `PreToolUse` instead
- `Stop` hooks fire whenever Claude finishes responding, not only at task completion; they do not fire on user interrupts

## Related concepts

- [Hook Types](./hook-types.md) — how each event's hook runs
- [Hook Matchers](./hook-matchers.md) — filtering which hooks fire
- [Hook Exit Codes](./hook-exit-codes.md) — communicating decisions back to Claude Code
- [Hook Scope](./hook-scope.md) — where hooks are configured
