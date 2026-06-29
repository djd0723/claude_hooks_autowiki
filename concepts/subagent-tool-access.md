---
type: concept
title: "Subagent Tool Access"
created: 2026-06-29
updated: 2026-06-29
tags: [subagents, tools, permissions, mcp, security]
source_count: 1
sources:
  - sources/clean/code-claude-com-docs-en-sub-agents-md.md
---

# Subagent Tool Access

You control what a [subagent](./subagents.md) can do through tool access, permission modes, and conditional rules.

## Available tools

Subagents inherit the internal tools and MCP tools available in the main conversation by default. The following depend on the main conversation's UI or session state and are **never** available to subagents, even when listed in `tools`:

- `AskUserQuestion`
- `EnterPlanMode`
- `ExitPlanMode` — unless the subagent's [`permissionMode`](#permission-modes) is `plan`
- `ScheduleWakeup`
- `WaitForMcpServers`

## Restricting tools: `tools` vs `disallowedTools`

Restrict with either an allowlist (`tools`) or a denylist (`disallowedTools`):

```yaml
# allowlist — only these four tools; no edits, writes, or MCP tools
tools: Read, Grep, Glob, Bash
```

```yaml
# denylist — inherit everything except Write and Edit
disallowedTools: Write, Edit
```

If both are set, `disallowedTools` is applied first, then `tools` is resolved against the remaining pool — a tool listed in both is removed. Both fields accept MCP server-level patterns alongside exact names: `mcp__<server>` or `mcp__<server>__*` grants or removes every tool from that server. In `disallowedTools`, `mcp__*` removes every MCP tool from any server.

## Restrict which subagents can be spawned

When an agent runs as the main thread with `claude --agent`, it can spawn subagents via the Agent tool. Restrict which types using `Agent(agent_type)` syntax in `tools`:

```yaml
tools: Agent(worker, researcher), Read, Bash
```

This is an allowlist — only `worker` and `researcher` can be spawned; other types fail and the agent sees only the allowed types. Use `Agent` without parentheses to allow spawning any subagent; omit `Agent` entirely to block spawning any.

> In version 2.1.63, the Task tool was renamed to Agent. Existing `Task(...)` references in settings and agent definitions still work as aliases.

The `Agent(agent_type)` allowlist applies only to an agent running as the main thread with `--agent`. In a subagent definition, listing `Agent` lets that subagent [spawn nested subagents](./subagent-context.md), but any type list inside the parentheses is ignored.

## Scope MCP servers to a subagent

Use `mcpServers` to give a subagent access to MCP servers that aren't in the main conversation. Each entry is either an inline server definition (same schema as `.mcp.json` — `stdio`, `http`, `sse`, `ws`) or a string referencing an already-configured server:

```yaml
mcpServers:
  - playwright:
      type: stdio
      command: npx
      args: ["-y", "@playwright/mcp@latest"]
  - github
```

Inline servers connect when the subagent starts and disconnect when it finishes; string references share the parent session's connection. To keep a server out of the main conversation entirely (so its tool descriptions don't consume context there), define it inline here rather than in `.mcp.json`.

As of v2.1.153, MCP restrictions that apply to the main session (`--strict-mcp-config`, `--bare`, enterprise managed MCP config, and `allowedMcpServers`/`deniedMcpServers` policies) also cover servers declared in subagent frontmatter; blocked servers are skipped with a warning. Managed-settings restrictions apply to every subagent regardless of how it's defined, but `--strict-mcp-config` doesn't filter servers passed inline via `--agents` or the SDK, since those are explicit caller input.

## Permission modes

The `permissionMode` field controls how the subagent handles permission prompts. Subagents inherit the [permission](./permission-settings.md) context from the main conversation and can override the mode, except where the parent takes precedence.

| Mode | Behavior |
| :--- | :------- |
| `default` | Standard permission checking with prompts |
| `acceptEdits` | Auto-accept file edits and common filesystem commands in the working directory or `additionalDirectories` |
| `auto` | A background classifier reviews commands and protected-directory writes |
| `dontAsk` | Auto-deny permission prompts (explicitly allowed tools still work) |
| `bypassPermissions` | Skip permission prompts |
| `plan` | Plan mode (read-only exploration) |

If the parent uses `bypassPermissions` or `acceptEdits`, that takes precedence and can't be overridden. If the parent uses auto mode, the subagent inherits it and any `permissionMode` in its frontmatter is ignored.

> Use `bypassPermissions` with caution. It skips permission prompts, allowing the subagent to execute operations without approval, including writes to `.git`, `.claude`, and similar directories. Explicit `ask` rules and root/home directory removals such as `rm -rf /` still prompt.

## Disable specific subagents

Prevent Claude from using specific subagents by adding them to the `deny` array in your [settings](./permission-settings.md), using `Agent(subagent-name)`:

```json
{ "permissions": { "deny": ["Agent(Explore)", "Agent(my-custom-agent)"] } }
```

This works for both built-in and custom subagents. You can also use `claude --disallowedTools "Agent(Explore)"`.

## Related concepts

- [Subagents](./subagents.md) — the core concept
- [Subagent Configuration](./subagent-configuration.md) — where `tools` and `mcpServers` are declared
- [Subagent Hooks](./subagent-hooks.md) — `PreToolUse` hooks for finer-grained conditional validation
- [Permission Settings](./permission-settings.md) — the `permissions.deny` rules used to disable subagents
