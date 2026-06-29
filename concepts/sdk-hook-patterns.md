---
type: concept
title: "SDK Hook Patterns"
created: 2026-06-29
updated: 2026-06-29
tags: [hooks, agent-sdk, patterns, examples, troubleshooting]
source_count: 1
sources:
  - sources/clean/code-claude-com-docs-en-agent-sdk-hooks-md.md
---

# SDK Hook Patterns

Common ways to use [SDK callback hooks](./sdk-callback-hooks.md), plus the failure modes that bite when they don't fire as expected. All examples are `PreToolUse` unless noted.

## Modify tool input

Rewrite the tool's arguments before execution by returning `updatedInput` together with `permissionDecision: "allow"`. The example redirects every `Write` to a `/sandbox` prefix:

```python Python
async def redirect_to_sandbox(input_data, tool_use_id, context):
    if input_data["hook_event_name"] != "PreToolUse":
        return {}
    if input_data["tool_name"] == "Write":
        original_path = input_data["tool_input"].get("file_path", "")
        return {
            "hookSpecificOutput": {
                "hookEventName": input_data["hook_event_name"],
                "permissionDecision": "allow",
                "updatedInput": {
                    **input_data["tool_input"],
                    "file_path": f"/sandbox{original_path}",
                },
            }
        }
    return {}
```

> When using `updatedInput`, you must also include `permissionDecision: 'allow'` to auto-approve the modified input or `permissionDecision: 'ask'` to show it to the user. With `'defer'`, `updatedInput` is ignored. Always return a new object rather than mutating the original `tool_input`.

## Block a tool and explain why

Return `permissionDecision: "deny"` to stop the call, `permissionDecisionReason` to tell the model why (so it avoids retrying), and `systemMessage` to show the user. The example blocks writes under `/etc`:

```python Python
async def block_etc_writes(input_data, tool_use_id, context):
    file_path = input_data["tool_input"].get("file_path", "")
    if file_path.startswith("/etc"):
        return {
            "systemMessage": "Remember: system directories like /etc are protected.",
            "hookSpecificOutput": {
                "hookEventName": input_data["hook_event_name"],
                "permissionDecision": "deny",
                "permissionDecisionReason": "Writing to /etc is not allowed",
            },
        }
    return {}
```

The canonical use is protecting `.env` files: check `tool_input.file_path` and return `permissionDecision: "deny"` when the filename matches.

## Auto-approve read-only tools

Skip permission prompts for safe tools by returning `permissionDecision: "allow"`, leaving all other tools subject to normal checks:

```python Python
read_only_tools = ["Read", "Glob", "Grep"]
if input_data["tool_name"] in read_only_tools:
    return {
        "hookSpecificOutput": {
            "hookEventName": input_data["hook_event_name"],
            "permissionDecision": "allow",
            "permissionDecisionReason": "Read-only tool auto-approved",
        }
    }
```

## Register multiple hooks and multi-tool matchers

Register several matchers under one event. Each runs in parallel; for permission decisions the most restrictive result wins (`deny > defer > ask > allow`). Mix scopes with the matcher field:

```python Python
options = ClaudeAgentOptions(
    hooks={
        "PreToolUse": [
            HookMatcher(matcher="Write|Edit|Delete", hooks=[file_security_hook]),  # exact list
            HookMatcher(matcher="^mcp__", hooks=[mcp_audit_hook]),                 # regex: all MCP tools
            HookMatcher(hooks=[global_logger]),                                    # no matcher: every tool
        ]
    }
)
```

> When an event fires, all matching hooks run in parallel. For permission decisions, the most restrictive result applies: a single `deny` blocks the tool call regardless of what the other hooks return. Because completion order is non-deterministic, write each hook to act independently rather than relying on another hook having run first.

## Side-effect hooks: HTTP, webhooks, Slack

Hooks can make HTTP requests. **Catch errors inside the hook** so a failed request never interrupts the agent, and in TypeScript pass the context `signal` so a request cancels if the hook times out:

```typescript TypeScript
const webhookNotifier: HookCallback = async (input, toolUseID, { signal }) => {
  if (input.hook_event_name !== "PostToolUse") return {};
  try {
    await fetch("https://api.example.com/webhook", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ tool: (input as PostToolUseHookInput).tool_name, timestamp: new Date().toISOString() }),
      signal
    });
  } catch (error) {
    if (error instanceof Error && error.name === "AbortError") console.log("Webhook request cancelled");
    // Don't re-throw. A failed webhook shouldn't stop the agent
  }
  return {};
};
```

For `Notification` hooks (forwarding agent status to Slack/PagerDuty), the same shape applies — run the blocking call in a thread (Python `asyncio.to_thread`) and return `{}`, since notification hooks don't modify behavior. Notifications fire for types such as `permission_prompt`, `idle_prompt`, `auth_success`, and the `elicitation_*` flows; each carries a `message` and optional `title`.

## Track subagent activity

`SubagentStop` callbacks log when a subagent finishes — `agent_id`, `agent_transcript_path`, the tool-use ID, and `stop_hook_active` are all on the input:

```python Python
options = ClaudeAgentOptions(
    hooks={"SubagentStop": [HookMatcher(hooks=[subagent_tracker])]}
)
```

## Troubleshooting

| Symptom | Likely cause / fix |
| :------ | :----------------- |
| Hook not firing | Event name is case-sensitive (`PreToolUse`, not `preToolUse`); matcher must match the tool name exactly; hook must be under the correct event key. Hooks may not fire when the agent hits `max_turns` — the session ends first |
| Matcher not filtering as expected | Matchers only match **tool names**, not file paths. Filter by path with `tool_input.file_path` inside the callback |
| Hook timeout | Increase `timeout` in the matcher; in TypeScript use the `AbortSignal` to cancel gracefully |
| Tool blocked unexpectedly | Some `PreToolUse` hook returned `permissionDecision: "deny"`; an empty matcher matches all tools. Log `permissionDecisionReason` to find which |
| Modified input not applied | `updatedInput` must be inside `hookSpecificOutput` (not top-level), with `permissionDecision: "allow"` (or `"ask"`) and `hookEventName` included |
| `systemMessage` not appearing | It shows to the user, not the model, and the SDK doesn't surface hook output by default. Set `includeHookEvents` (`include_hook_events` in Python); to pass context to the model, return `additionalContext` instead |
| Subagent permission prompts multiplying | Subagents don't inherit parent permissions. Auto-approve with `PreToolUse` hooks or configure permission rules for subagent sessions |
| Recursive hook loops | A `UserPromptSubmit` hook that spawns subagents can loop if those subagents trigger the same hook. Check for a subagent indicator before spawning, or scope the hook to the top-level session only |

## Related concepts

- [SDK Callback Hooks](./sdk-callback-hooks.md) — registration, callback signature, outputs
- [SDK Hook Events](./sdk-hook-events.md) — Python vs. TypeScript availability
- [Hook Decision Control](./hook-decision-control.md) — the full decision-pattern reference
- [Hook Matchers](./hook-matchers.md) — matcher syntax (shared with the SDK)
- [Permission Settings](./permission-settings.md) — permission rules that interact with hook decisions
