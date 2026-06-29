---
type: concept
title: "Subagent Hooks"
created: 2026-06-29
updated: 2026-06-29
tags: [subagents, hooks, lifecycle, validation, automation]
source_count: 1
sources:
  - sources/clean/code-claude-com-docs-en-sub-agents-md.md
---

# Subagent Hooks

[Subagents](./subagents.md) can define [hooks](./hook-lifecycle-events.md) that run during their lifecycle. There are two ways to configure them: in the subagent's own frontmatter (scoped to that subagent), or in `settings.json` (run in the main session when subagents start or stop).

## Hooks in subagent frontmatter

Define hooks directly in the subagent's markdown file. These run only while that specific subagent is active and are cleaned up when it finishes. They fire both when the agent is spawned as a subagent (through the Agent tool or an @-mention) and when it runs as the main session via `--agent` or the `agent` setting.

All [hook events](./hook-lifecycle-events.md) are supported. The most common for subagents:

| Event | Matcher input | When it fires |
| :---- | :------------ | :------------ |
| `PreToolUse` | Tool name | Before the subagent uses a tool |
| `PostToolUse` | Tool name | After the subagent uses a tool |
| `Stop` | (none) | When the subagent finishes (converted to `SubagentStop` at runtime) |

```yaml
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "./scripts/validate-command.sh $TOOL_INPUT"
  PostToolUse:
    - matcher: "Edit|Write"
      hooks:
        - type: command
          command: "./scripts/run-linter.sh"
```

When the agent is invoked as a subagent, `Stop` hooks in frontmatter are automatically converted to `SubagentStop` events.

> [Plugin subagents](./subagent-configuration.md) don't support the `hooks` field — it is ignored when loading agents from a plugin.

## Conditional rules with PreToolUse

For dynamic control over tool usage, use `PreToolUse` hooks to validate operations before they execute — useful when you need to allow some operations of a tool while blocking others. This example permits only read-only database queries:

```yaml
---
name: db-reader
description: Execute read-only database queries
tools: Bash
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "./scripts/validate-readonly-query.sh"
---
```

Claude Code [passes hook input as JSON](./hook-input-output.md) via stdin. The validation script reads it, extracts the Bash command, and [exits with code 2](./hook-exit-codes.md) to block write operations:

```bash
#!/bin/bash
INPUT=$(cat)
COMMAND=$(echo "$INPUT" | jq -r '.tool_input.command // empty')
if echo "$COMMAND" | grep -iE '\b(INSERT|UPDATE|DELETE|DROP|CREATE|ALTER|TRUNCATE)\b' > /dev/null; then
  echo "Blocked: Only SELECT queries are allowed" >&2
  exit 2
fi
exit 0
```

On Windows, write hook scripts in PowerShell and add `shell: powershell` to the hook entry.

## Project-level hooks for subagent events

Configure hooks in `settings.json` that respond to subagent lifecycle events in the main session:

| Event | Matcher input | When it fires |
| :---- | :------------ | :------------ |
| `SubagentStart` | Agent type name | When a subagent begins execution |
| `SubagentStop` | Agent type name | When a subagent completes |

Both support [matchers](./hook-matchers.md) to target specific agent types by name. This example runs a setup script only when the `db-agent` subagent starts, and a cleanup script when any subagent stops:

```json
{
  "hooks": {
    "SubagentStart": [
      { "matcher": "db-agent", "hooks": [{ "type": "command", "command": "./scripts/setup-db-connection.sh" }] }
    ],
    "SubagentStop": [
      { "hooks": [{ "type": "command", "command": "./scripts/cleanup-db-connection.sh" }] }
    ]
  }
}
```

The agent type matched here is the subagent's [`name` field](./subagent-configuration.md).

## Related concepts

- [Subagents](./subagents.md) — the core concept
- [Subagent Configuration](./subagent-configuration.md) — the `hooks` frontmatter field and the `name` used as `agent_type`
- [Subagent Tool Access](./subagent-tool-access.md) — pairs `PreToolUse` validation with `tools`/`disallowedTools`
- [Hook Lifecycle Events](./hook-lifecycle-events.md) — `SubagentStart` / `SubagentStop` in the full event catalog
- [Hook Input and Output](./hook-input-output.md) — the JSON envelope passed to hook commands
- [Hook Exit Codes](./hook-exit-codes.md) — exit 2 to block an operation
