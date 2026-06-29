---
type: comparison
title: "Monitors vs command hooks"
created: 2026-06-29
updated: 2026-06-29
tags: [monitors, hooks, command, background, async, notifications, plugins]
sources:
  - sources/clean/code-claude-com-docs-en-plugins-reference-md.md
  - sources/clean/code-claude-com-docs-en-hooks-md.md
---

# Monitors vs command hooks

Both monitors and command hooks run shell commands in the context of a Claude Code session.
They diverge fundamentally in trigger model, lifecycle, and how they communicate with Claude.

## Side-by-side comparison

| Dimension | Monitor | Command hook |
| :-------- | :------ | :----------- |
| Trigger | Continuous — runs as a persistent background process | Event-driven — fires on a specific lifecycle event (e.g., `PostToolUse`) |
| Lifecycle | Starts at session start (or skill invoke) and runs until the session ends | Runs once per event occurrence; each invocation is a fresh process |
| Communication channel | Every stdout line is delivered to Claude as a notification | Exit code and JSON on stdout/stderr; can also use `asyncRewake` to notify Claude |
| Can block or gate an action | No — monitors have no decision output | Yes (exit code 2 on sync hooks); async hooks cannot block |
| Filtering | `when` field: `"always"` or `"on-skill-invoke:<skill-name>"` | `matcher` regex on tool name; event type selection |
| Session awareness | Tied to session lifetime; not stopped if plugin disabled mid-session | Stateless per invocation; no built-in session state |
| Config location | `monitors/monitors.json` or `plugin.json` `experimental.monitors` | `hooks/hooks.json`, `plugin.json` `hooks`, or settings JSON |
| Availability | Interactive CLI sessions only; not loaded for project-scope `@skills-dir` plugins | All sessions including non-interactive (`-p` flag); some events skip `PermissionRequest` in non-interactive mode |
| Minimum version | Claude Code v2.1.105+ | All versions |

## When to use each

**Use a monitor** when Claude needs to react to a continuous external stream — log tailing, deploy
status polling, file-system watchers, CI pipelines. Monitors push information to Claude on their own
schedule, decoupled from any specific Claude action.

```json
{ "name": "error-log", "command": "tail -F ./logs/error.log", "description": "Application error log" }
```

**Use a command hook** when Claude's own actions should trigger a check, side effect, or gate. Hooks
couple to the lifecycle of what Claude is doing: formatting after an edit, enforcing a policy before
a tool runs, logging after a session ends.

```json
{ "type": "command", "command": "npx prettier --write \"$TOOL_ARG_FILE_PATH\"" }
```

## The `asyncRewake` middle ground

`asyncRewake: true` on a command hook is the closest hooks get to monitor behavior: the hook runs in
the background after the action, and if it exits with code 2, Claude resumes and receives the hook's
stderr as a system reminder. Unlike monitors, `asyncRewake` hooks are still single-shot and
event-triggered — they don't persist between events.

```json
{ "type": "command", "asyncRewake": true, "command": ".claude/hooks/verify-ci.sh" }
```

## Capability restrictions

Monitors are the more restricted primitive:
- Require interactive CLI mode (skipped on hosts without the Monitor tool)
- Not loaded for project-scope `@skills-dir` plugins
- Cannot block or influence actions

Command hooks face fewer restrictions: they run in both interactive and non-interactive sessions, and
sync hooks can gate any action (even in `bypassPermissions` mode — `PreToolUse` hooks always fire
first).

## Related concepts

- [Plugin Components](../concepts/plugin-components.md) — monitors as one of seven plugin component types
- [Hook Types](../concepts/hook-types.md) — full field reference for command hooks including `async` and `asyncRewake`
- [Hook Exit Codes](../concepts/hook-exit-codes.md) — how command hooks communicate decisions
- [Plugin Installation Scopes](../concepts/plugin-installation-scopes.md) — the project-scope restriction on monitors
