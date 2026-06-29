---
type: concept
title: "SDK Hook Event Availability"
created: 2026-06-29
updated: 2026-06-29
tags: [hooks, agent-sdk, python, typescript, events, parity]
source_count: 1
sources:
  - sources/clean/code-claude-com-docs-en-agent-sdk-hooks-md.md
---

# SDK Hook Event Availability

The Agent SDK exposes hooks for different stages of agent execution. Some are available in both the Python and TypeScript SDKs; others are TypeScript-only. This page is the parity reference — for the full CLI event list and cadences, see [Hook Lifecycle Events](./hook-lifecycle-events.md).

## Availability matrix

| Hook Event | Python SDK | TypeScript SDK | What triggers it | Example use case |
| :--------- | :--------: | :------------: | :--------------- | :--------------- |
| `PreToolUse` | Yes | Yes | Tool call request (can block or modify) | Block dangerous shell commands |
| `PostToolUse` | Yes | Yes | Tool execution result | Log all file changes to audit trail |
| `PostToolUseFailure` | Yes | Yes | Tool execution failure | Handle or log tool errors |
| `PostToolBatch` | No | Yes | A full batch of tool calls resolves, once per batch before the next model call | Inject conventions once for the whole batch |
| `UserPromptSubmit` | Yes | Yes | User prompt submission | Inject additional context into prompts |
| `MessageDisplay` | No | Yes | An assistant message with text completes, once per message | Redact or reformat displayed text without changing the transcript |
| `Stop` | Yes | Yes | Agent execution stop | Save session state before exit |
| `SubagentStart` | Yes | Yes | Subagent initialization | Track parallel task spawning |
| `SubagentStop` | Yes | Yes | Subagent completion | Aggregate results from parallel tasks |
| `PreCompact` | Yes | Yes | Conversation compaction request | Archive full transcript before summarizing |
| `PermissionRequest` | Yes | Yes | Permission dialog would be displayed | Custom permission handling |
| `SessionStart` | No | Yes | Session initialization | Initialize logging and telemetry |
| `SessionEnd` | No | Yes | Session termination | Clean up temporary resources |
| `Notification` | Yes | Yes | Agent status messages | Send status updates to Slack or PagerDuty |
| `Setup` | No | Yes | Session setup/maintenance | Run initialization tasks |
| `TeammateIdle` | No | Yes | Teammate becomes idle | Reassign work or notify |
| `TaskCompleted` | No | Yes | Background task completes | Aggregate results from parallel tasks |
| `ConfigChange` | No | Yes | Configuration file changes | Reload settings dynamically |
| `WorktreeCreate` | No | Yes | Git worktree created | Track isolated workspaces |
| `WorktreeRemove` | No | Yes | Git worktree removed | Clean up workspace resources |

## Python vs. TypeScript parity

The events shared by both SDKs are the core tool/lifecycle set: `PreToolUse`, `PostToolUse`, `PostToolUseFailure`, `UserPromptSubmit`, `Stop`, `SubagentStart`, `SubagentStop`, `PreCompact`, `PermissionRequest`, and `Notification`.

Everything else in the table is **TypeScript-only** as an SDK callback: batch, display, session-lifecycle, setup, teammate, task, config, and worktree events.

### Session hooks in Python

`SessionStart` and `SessionEnd` aren't available as Python SDK callback hooks because the Python `HookEvent` type omits them. In Python they are only available as [shell-command hooks](./hook-types.md) defined in settings files such as `.claude/settings.json`. To load those from an SDK application, include the appropriate setting source:

```python Python
options = ClaudeAgentOptions(
    setting_sources=["project"],  # Loads .claude/settings.json including hooks
)
```

```typescript TypeScript
const options = {
  settingSources: ["project"] // Loads .claude/settings.json including hooks
};
```

To run initialization logic as a Python SDK callback instead, use the first message from `client.receive_response()` as your trigger.

## Related concepts

- [SDK Callback Hooks](./sdk-callback-hooks.md) — registering and writing the callbacks
- [SDK Hook Patterns](./sdk-hook-patterns.md) — worked examples and troubleshooting
- [Hook Lifecycle Events](./hook-lifecycle-events.md) — the full CLI event list and cadences
- [Hook Types](./hook-types.md) — command, http, mcp_tool, prompt, agent hook types
- [Subagent Hooks](./subagent-hooks.md) — `SubagentStart` / `SubagentStop` details
