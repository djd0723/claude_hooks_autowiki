---
type: concept
title: "SDK Callback Hooks"
created: 2026-06-29
updated: 2026-06-29
tags: [hooks, agent-sdk, python, typescript, callbacks, automation]
source_count: 1
sources:
  - sources/clean/code-claude-com-docs-en-agent-sdk-hooks-md.md
---

# SDK Callback Hooks

The Agent SDK lets you register hooks as in-process **callback functions** instead of (or alongside) the shell-command hooks defined in settings files. A callback runs your code in response to agent events — a tool being called, a session starting, execution stopping.

> Hooks are callback functions that run your code in response to agent events, like a tool being called, a session starting, or execution stopping.

These are the same hook events documented for the CLI (see [Hook Lifecycle Events](./hook-lifecycle-events.md)), surfaced as language-native callbacks you pass in `options.hooks`.

## How a hook fires

1. **An event fires** — something happens during agent execution (`PreToolUse`, `PostToolUse`, a subagent start/stop, idle, finish).
2. **The SDK collects registered hooks** — both callback hooks from `options.hooks` and shell-command hooks from settings files when the corresponding `settingSources` / `setting_sources` entry is enabled (which it is for default `query()` options).
3. **Matchers filter which hooks run** — a hook with a [`matcher`](./hook-matchers.md) pattern like `"Write|Edit"` is tested against the event's target (e.g. the tool name); hooks without a matcher run for every event of that type.
4. **Callback functions execute** — each matching callback receives input about what's happening: the tool name, its arguments, the session ID, and other event-specific details.
5. **Your callback returns a decision** — an output object that tells the agent what to do: allow, block, modify the input, or inject context.

## Registration

The `hooks` option is a dictionary in Python or an object in TypeScript. **Keys** are hook event names (`'PreToolUse'`, `'PostToolUse'`, `'Stop'`, …); **values** are arrays of matchers, each pairing an optional filter pattern with your callbacks.

```python Python
options = ClaudeAgentOptions(
    hooks={"PreToolUse": [HookMatcher(matcher="Bash", hooks=[my_callback])]}
)

async with ClaudeSDKClient(options=options) as client:
    await client.query("Your prompt")
    async for message in client.receive_response():
        print(message)
```

```typescript TypeScript
for await (const message of query({
  prompt: "Your prompt",
  options: {
    hooks: {
      PreToolUse: [{ matcher: "Bash", hooks: [myCallback] }]
    }
  }
})) {
  console.log(message);
}
```

Python wraps each matcher in a `HookMatcher(matcher=…, hooks=[…])`; TypeScript uses a plain object `{ matcher: …, hooks: […] }`. A matcher group also accepts a `timeout` (seconds, default `60`). For matcher syntax — exact-string vs. regex, `mcp__<server>__<action>`, `*`/empty/omitted meaning match-all — the SDK follows the same rules as [Hook Matchers](./hook-matchers.md).

## Callback signature

Every hook callback receives three arguments:

- **Input data** — a typed object of event details. Each hook type has its own shape: `PreToolUseHookInput` includes `tool_name` and `tool_input`; `NotificationHookInput` includes `message`. All inputs share `session_id`, `cwd`, and `hook_event_name`. `agent_id` / `agent_type` are populated when the hook fires inside a subagent (in TypeScript on every hook type; in Python only on `PreToolUse`, `PostToolUse`, and `PostToolUseFailure`).
- **Tool use ID** (`str | None` / `string | undefined`) — correlates `PreToolUse` and `PostToolUse` events for the same tool call.
- **Context** — in TypeScript, contains a `signal` (`AbortSignal`) for cancellation. In Python this argument is reserved for future use.

## Outputs

A callback returns an object with two categories of fields:

- **Top-level fields** behave the same on every event: `systemMessage` shows a message to the user, and `continue` (`continue_` in Python) decides whether the agent keeps running after the hook.
- **`hookSpecificOutput`** controls the current operation; its fields depend on the event. For `PreToolUse`, this is where you set `permissionDecision` (`"allow"`, `"deny"`, `"ask"`, or `"defer"`), `permissionDecisionReason`, and `updatedInput`. For `PostToolUse`, `additionalContext` appends to the tool result and `updatedToolOutput` replaces the tool's output before Claude sees it (works for any tool in both SDKs; the older `updatedMCPToolOutput` is deprecated).

Return `{}` to allow the operation without changes. SDK callback hooks use the **same JSON output format** as Claude Code shell-command hooks — see [Hook Input and Output](./hook-input-output.md) and [Hook Decision Control](./hook-decision-control.md) for the full field set.

> When multiple hooks or permission rules apply, `deny` takes priority over `defer`, which takes priority over `ask`, which takes priority over `allow`. If any hook returns `deny`, the operation is blocked regardless of other hooks.

When an event fires, all matching hooks run **in parallel**; completion order is non-deterministic, so write each hook to act independently rather than relying on another having run first.

## Asynchronous output

By default the agent waits for your hook to return. For pure side effects — logging, webhooks — return an async output so the agent continues immediately:

```python Python
async def async_hook(input_data, tool_use_id, context):
    asyncio.create_task(send_to_logging_service(input_data))
    return {"async_": True, "asyncTimeout": 30000}
```

```typescript TypeScript
const asyncHook: HookCallback = async (input, toolUseID, { signal }) => {
  sendToLoggingService(input).catch(console.error);
  return { async: true, asyncTimeout: 30000 };
};
```

`async` (Python: `async_`, to avoid the reserved keyword) signals async mode; `asyncTimeout` is an optional millisecond timeout.

> Async outputs can't block, modify, or inject context into the operation since the agent has already moved on. Use them only for side effects like logging, metrics, or notifications.

## Related concepts

- [SDK Setting Sources](./sdk-setting-sources.md) — how filesystem hooks are loaded alongside these callbacks
- [SDK Hook Events](./sdk-hook-events.md) — which events each SDK supports (Python vs. TypeScript)
- [SDK Hook Patterns](./sdk-hook-patterns.md) — worked examples and troubleshooting
- [Hook Lifecycle Events](./hook-lifecycle-events.md) — the full event list
- [Hook Matchers](./hook-matchers.md) — matcher syntax (shared with the SDK)
- [Hook Input and Output](./hook-input-output.md) — the shared JSON output schema
- [Hook Decision Control](./hook-decision-control.md) — per-event decision patterns
- [Subagent Hooks](./subagent-hooks.md) — `SubagentStart` / `SubagentStop` behavior
