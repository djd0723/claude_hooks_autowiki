---
type: comparison
title: "Synchronous vs async hook execution"
created: 2026-06-29
updated: 2026-06-29
tags: [hooks, async, command, blocking, background, asyncRewake]
sources:
  - sources/clean/code-claude-com-docs-en-hooks-md.md
---

# Synchronous vs async hook execution

Command hooks run synchronously by default — Claude waits for the hook to exit before proceeding. Adding `async: true` or `asyncRewake: true` moves the hook to the background. The choice determines whether the hook can gate the action and whether failures surface back to Claude.

## Comparison

| Dimension | Sync (default) | `async: true` | `asyncRewake: true` |
| :-------- | :------------- | :------------ | :------------------ |
| Claude waits for hook | Yes | No | No |
| Can block or deny the action | Yes (exit code 2) | No — decision fields ignored, action already completed | No — only wakes Claude after the fact |
| Exit code 2 behavior | Blocks the action (or wakes Claude if event supports it) | Ignored | Wakes Claude with hook stderr/stdout as system reminder |
| Errors shown in transcript | Yes (non-0, non-2: hook error notice in transcript) | No | Yes (on exit 2 only, as system reminder) |
| Use for gating / enforcement | Yes — the primary use case | No | No |
| Use for side effects | Works, but holds up Claude | Yes — ideal | Yes — ideal for long-running background work that may need Claude's attention on failure |
| Subject to event timeout | Yes (default 10 min; lower for some events) | No | No |

## When to use which

**Synchronous** (default): use whenever the hook must influence the outcome — formatters, policy checkers, pre-commit validators, permission gatekeepers. The hook's exit code and JSON output are the communication channel.

```json
{ "type": "command", "command": "npx prettier --write \"$TOOL_ARG_FILE_PATH\"" }
```

**`async: true`**: use for side effects that don't need to change the outcome — audit logging, metrics emission, notifications, CI triggers. The action completes without waiting for the hook.

```json
{ "type": "command", "async": true, "command": "audit-log.sh" }
```

> "Decision output fields (`decision`, `permissionDecision`, `continue`) are ignored [for async hooks] — the action has already completed."

**`asyncRewake: true`**: use for long-running background verification that might fail in a way Claude should react to. The hook runs after the action; if it exits with code 2, Claude resumes and receives the hook's stderr (or stdout if stderr is empty) as a system reminder, allowing it to take corrective action.

```json
{ "type": "command", "asyncRewake": true, "command": ".claude/hooks/verify-ci.sh" }
```

`asyncRewake` is the right choice when you want background work but need a safety net: the hook can't stop the action, but it can bring a problem to Claude's attention after the fact.

## Key constraint

`async` and `asyncRewake` only apply to `"type": "command"` hooks. HTTP, prompt, and agent hooks are always synchronous.

## Related concepts

- [Hook Types](../concepts/hook-types.md) — full field reference for command hooks
- [Hook Exit Codes](../concepts/hook-exit-codes.md) — how exit codes communicate decisions
- [Hook Decision Control](../concepts/hook-decision-control.md) — per-event decision patterns
