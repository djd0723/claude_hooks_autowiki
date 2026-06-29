---
type: summary
title: "Agent SDK hooks — callback hooks in code"
slug: code-claude-com-docs-en-agent-sdk-hooks-md
created: 2026-06-29
updated: 2026-06-29
tags: [hooks, agent-sdk, callbacks, typescript, python, decision-control]
sources:
  - sources/clean/code-claude-com-docs-en-agent-sdk-hooks-md.md
---

# Summary: Agent SDK hooks

How to intercept and control agent behavior with in-process callback hooks registered through the Claude Agent SDK (TypeScript and Python). This is the SDK-side companion to the [hooks reference summary](hooks.md): both share the same JSON output format and matcher rules, but here hooks are JavaScript/Python functions passed in code rather than shell command hooks declared in settings files.

## Key thesis from source

> "Hooks are callback functions that run your code in response to agent events, like a tool being called, a session starting, or execution stopping."

The guide frames five use cases: blocking dangerous operations, logging/auditing tool calls, transforming inputs and outputs, requiring human approval, and tracking session lifecycle.

## How SDK hooks differ from CLI command hooks

The CLI/settings-file hooks in the [hooks reference](hooks.md) run external shell commands that communicate over stdin/stdout JSON. SDK hooks are **callback functions executing in your process**:

- Registered in code via `options.hooks` (`ClaudeAgentOptions` in Python, the `options` object in TypeScript), keyed by event name, with values being arrays of matchers each holding a `matcher` pattern and `hooks` callbacks.
- Each callback receives typed input objects, returns an output object (no serialization to write by hand).
- The SDK *also* picks up shell command hooks from settings files when the matching `settingSources` / `setting_sources` entry is enabled (the default for `query()`), so both kinds can coexist.

The five-step lifecycle: an event fires, the SDK collects registered hooks (callback + settings), matchers filter which run, callbacks execute receiving event input, and each callback returns a decision (allow/block/modify/inject context).

## Available hook events

Some events exist in both SDKs; others are TypeScript-only.

| Event | Python | TS | Trigger |
| :---- | :----- | :- | :------ |
| `PreToolUse` | Yes | Yes | Tool call request (can block or modify) |
| `PostToolUse` | Yes | Yes | Tool execution result |
| `PostToolUseFailure` | Yes | Yes | Tool execution failure |
| `PostToolBatch` | No | Yes | A full batch of tool calls resolves |
| `UserPromptSubmit` | Yes | Yes | User prompt submission |
| `MessageDisplay` | No | Yes | Assistant text message completes |
| `Stop` | Yes | Yes | Agent execution stop |
| `SubagentStart` | Yes | Yes | Subagent initialization |
| `SubagentStop` | Yes | Yes | Subagent completion |
| `PreCompact` | Yes | Yes | Conversation compaction request |
| `PermissionRequest` | Yes | Yes | Permission dialog would display |
| `SessionStart` | No | Yes | Session initialization |
| `SessionEnd` | No | Yes | Session termination |
| `Notification` | Yes | Yes | Agent status messages |
| `Setup` | No | Yes | Session setup/maintenance |
| `TeammateIdle` | No | Yes | Teammate becomes idle |
| `TaskCompleted` | No | Yes | Background task completes |
| `ConfigChange` | No | Yes | Configuration file changes |
| `WorktreeCreate` | No | Yes | Git worktree created |
| `WorktreeRemove` | No | Yes | Git worktree removed |

See [SDK hook events](../concepts/sdk-hook-events.md) for the Python/TypeScript availability split.

## Matchers

SDK matchers follow the same comparison rules as settings-file matchers (see [hook matchers](../concepts/hook-matchers.md) via the reference). A matcher of only letters/digits/`_`/spaces/`,`/`|` is an exact or OR-list match (`Write|Edit` or `Write, Edit`); `*`, empty, or omitted matches every event; anything else is a regex (`^mcp__` matches all MCP tools).

| Option | Type | Default | Description |
| :----- | :--- | :------ | :---------- |
| `matcher` | `string` | `undefined` | Pattern matched against the event's filter field (tool name for tool hooks) |
| `hooks` | `HookCallback[]` | — | Required. Callbacks to run when the pattern matches |
| `timeout` | `number` | `60` | Timeout in seconds |

Matchers filter by tool name only, not file paths. To filter by path, check `tool_input.file_path` inside the callback. MCP tools use `mcp__<server>__<action>`; note `mcp__memory` is an exact (no-match) string — use `mcp__memory__.*` for a server's tools.

## Callback signatures

Every callback receives three arguments:

1. **Input data** — a typed, event-specific object. All inputs share `session_id`, `cwd`, and `hook_event_name`. `agent_id`/`agent_type` are present when firing inside a subagent (on the base input in TS; only on `PreToolUse`/`PostToolUse`/`PostToolUseFailure` in Python).
2. **Tool use ID** (`str | None` / `string | undefined`) — correlates `PreToolUse` and `PostToolUse` for the same call.
3. **Context** — in TS, has a `signal` (`AbortSignal`) for cancellation; in Python, reserved for future use.

## Return shapes (outputs)

Callbacks return an object with two field categories. Return `{}` to allow without changes. The format matches [Claude Code shell command hooks JSON output](hooks.md).

- **Top-level fields** (every event): `systemMessage` (message to the user), `continue` / `continue_` (whether the agent keeps running).
- **`hookSpecificOutput`** (per-event): for `PreToolUse`, set `permissionDecision` (`"allow"`/`"deny"`/`"ask"`/`"defer"`), `permissionDecisionReason`, and `updatedInput`. For `PostToolUse`, set `additionalContext` to append to the tool result, or `updatedToolOutput` to replace output before Claude sees it (the older `updatedMCPToolOutput` is deprecated).

> "When multiple hooks or permission rules apply, `deny` takes priority over `defer`, which takes priority over `ask`, which takes priority over `allow`. If any hook returns `deny`, the operation is blocked regardless of other hooks."

Using `updatedInput` requires `permissionDecision: 'allow'` (auto-approve) or `'ask'` (show user); with `'defer'`, `updatedInput` is ignored. Always return a new object rather than mutating `tool_input`.

### Synchronous vs asynchronous output

By default the agent waits for the callback. For pure side effects (logging, webhooks), return an async output so the agent continues immediately — see [sync vs async hooks](../comparisons/sync-vs-async-hooks.md).

| Field | Type | Description |
| :---- | :--- | :---------- |
| `async` (`async_` in Python) | `true` | Signals async mode; agent proceeds without waiting |
| `asyncTimeout` | `number` | Optional timeout in milliseconds for the background work |

> "Async outputs can't block, modify, or inject context into the operation since the agent has already moved on."

## Example patterns

The guide demonstrates these patterns (detailed in [SDK hook patterns](../concepts/sdk-hook-patterns.md)):

- **Block by path** — `PreToolUse` with matcher `"Write|Edit"` returns `permissionDecision: "deny"` for `.env` files.
- **Modify tool input** — rewrite `file_path` to prepend `/sandbox`, returning `updatedInput` plus `permissionDecision: "allow"`.
- **Add context and block** — `deny` plus `permissionDecisionReason` (to the model) plus `systemMessage` (to the user) for `/etc` writes.
- **Auto-approve** read-only tools (`Read`, `Glob`, `Grep`).
- **Register multiple hooks** — all matching hooks run in parallel; most restrictive decision wins; write each to act independently.
- **Multi-tool matchers** — exact OR-list, `^mcp__` regex, and an omitted (match-all) matcher in one config.
- **Track subagents** — `SubagentStop` callback logs `agent_id`, `agent_transcript_path`, `tool_use_id` (see [subagent hooks](../concepts/subagent-hooks.md)).
- **HTTP requests / Slack** — `PostToolUse` and `Notification` callbacks POST to webhooks, catching errors internally so failures don't interrupt the agent. `Notification` fires for `permission_prompt`, `idle_prompt`, `auth_success`, and elicitation events, each with a `message` field.

## Common issues

- Event names are case-sensitive (`PreToolUse`, not `preToolUse`).
- Matchers match tool names only — filter paths inside the callback.
- Hooks may not fire when the agent hits `max_turns` (session ends first).
- `systemMessage` targets the user, not the model, and may not surface unless `includeHookEvents` / `include_hook_events` is set — use `additionalContext` to reach the model.
- `SessionStart`/`SessionEnd` aren't Python callback hooks (its `HookEvent` omits them); register them as shell command hooks in settings and load via `setting_sources`, or use the first `client.receive_response()` message as an init trigger.
- Subagents don't inherit parent permissions (prompts can multiply); a `UserPromptSubmit` hook that spawns subagents can recurse — guard with a subagent indicator or session state.

## Relationship to concept pages

This summary is the SDK-side counterpart to the [hooks reference summary](hooks.md). It backs:

- [SDK hook events](../concepts/sdk-hook-events.md) — the event table and Python/TypeScript availability
- [SDK callback hooks](../concepts/sdk-callback-hooks.md) — callback registration, signatures, and return shapes
- [SDK hook patterns](../concepts/sdk-hook-patterns.md) — block/modify/approve/log example patterns
- [Subagent hooks](../concepts/subagent-hooks.md) — `SubagentStart`/`SubagentStop` and subagent input fields
- [Sync vs async hooks](../comparisons/sync-vs-async-hooks.md) — when to return async output
- [SDK extension features](../comparisons/sdk-extension-features.md) — hooks alongside other SDK customization points
