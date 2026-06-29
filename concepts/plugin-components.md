---
type: concept
title: "Plugin Components"
created: 2026-06-29
updated: 2026-06-29
tags: [plugins, components, skills, agents, hooks, mcp, lsp, monitors, themes]
source_count: 1
sources:
  - sources/clean/code-claude-com-docs-en-plugins-reference-md.md
---

# Plugin Components

A plugin packages one or more components that extend Claude Code. The seven component types have distinct purposes, default locations, and integration behaviors.

## Skills

**Default location**: `skills/` (named subdirectories with `SKILL.md`) or `commands/` (flat `.md` files)

Skills create `/name` shortcuts that users or Claude can invoke. Automatically discovered on plugin install.

```text
skills/
├── pdf-processor/
│   ├── SKILL.md
│   └── scripts/
└── code-reviewer/
    └── SKILL.md
```

A `SKILL.md` at the plugin root (with no `skills/` directory and no `skills` manifest field) is loaded as a single-skill plugin (Claude Code v2.1.142+). The frontmatter `name` field controls the invocation name; without it, the install directory name is used — which changes on every marketplace update.

## Agents

**Default location**: `agents/` directory

Plugin agents are specialized subagents Claude invokes automatically based on task context. They appear in the `/agents` interface.

```markdown
---
name: agent-name
description: When Claude should invoke this agent and what it specializes in
model: sonnet
effort: medium
maxTurns: 20
disallowedTools: Write, Edit
---

System prompt describing the agent's role and expertise.
```

Supported frontmatter: `name`, `description`, `model`, `effort`, `maxTurns`, `tools`, `disallowedTools`, `skills`, `memory`, `background`, `isolation` (only `"worktree"` is valid).

Security restriction: `hooks`, `mcpServers`, and `permissionMode` are **not** supported for plugin-shipped agents.

## Hooks

**Default location**: `hooks/hooks.json`, or inline in `plugin.json`

Plugin hooks respond to the same 30+ lifecycle events as user-defined hooks. See [Hook Lifecycle Events](./hook-lifecycle-events.md) for the full event table and [Hook Types](./hook-types.md) for `command`, `http`, `mcp_tool`, `prompt`, and `agent` hook types.

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "\"${CLAUDE_PLUGIN_ROOT}\"/scripts/format-code.sh"
          }
        ]
      }
    ]
  }
}
```

## MCP Servers

**Default location**: `.mcp.json` in plugin root, or inline in `plugin.json`

Plugin MCP servers start automatically when the plugin is enabled. They appear as standard MCP tools in Claude's toolkit, configured independently of user MCP servers.

```json
{
  "mcpServers": {
    "plugin-database": {
      "command": "${CLAUDE_PLUGIN_ROOT}/servers/db-server",
      "args": ["--config", "${CLAUDE_PLUGIN_ROOT}/config.json"],
      "env": { "DB_PATH": "${CLAUDE_PLUGIN_ROOT}/data" }
    }
  }
}
```

## LSP Servers

**Default location**: `.lsp.json` in plugin root, or inline in `plugin.json`

LSP servers give Claude real-time code intelligence: instant diagnostics after each edit, go-to-definition, find-references, and hover information. The LSP binary must be installed separately — the plugin only configures the connection.

```json
{
  "go": {
    "command": "gopls",
    "args": ["serve"],
    "extensionToLanguage": { ".go": "go" }
  }
}
```

Required fields: `command`, `extensionToLanguage`.

Optional fields: `args`, `transport`, `env`, `initializationOptions`, `settings`, `workspaceFolder`, `startupTimeout`, `maxRestarts`, `diagnostics` (set to `false` to suppress automatic diagnostic injection while keeping code navigation).

Official marketplace LSP plugins: `pyright-lsp` (Python), `typescript-lsp` (TypeScript), `rust-analyzer-lsp` (Rust).

## Monitors

**Default location**: `monitors/monitors.json`, or inline in `plugin.json` as `experimental.monitors`. Requires Claude Code v2.1.105+.

Monitors are persistent background shell commands. Every stdout line is delivered to Claude as a notification, so Claude can react to log entries and status changes without being prompted.

```json
[
  {
    "name": "deploy-status",
    "command": "\"${CLAUDE_PLUGIN_ROOT}\"/scripts/poll-deploy.sh ${user_config.api_endpoint}",
    "description": "Deployment status changes"
  },
  {
    "name": "error-log",
    "command": "tail -F ./logs/error.log",
    "description": "Application error log",
    "when": "on-skill-invoke:debug"
  }
]
```

Required fields: `name`, `command`, `description`.

Optional `when` field:
- `"always"` (default) — starts at session start and on plugin reload
- `"on-skill-invoke:<skill-name>"` — starts the first time the named skill in this plugin is dispatched

**Restrictions**: run only in interactive CLI sessions; skipped on hosts where the Monitor tool is unavailable; unsandboxed at the same trust level as hooks; not loaded for project-scope `@skills-dir` plugins.

Disabling a plugin mid-session does not stop monitors already running — they stop when the session ends.

## Themes

**Default location**: `themes/` directory. Experimental component.

Themes ship color schemes that appear in `/theme` alongside built-in presets and the user's local themes.

```json
{
  "name": "Dracula",
  "base": "dark",
  "overrides": {
    "claude": "#bd93f9",
    "error": "#ff5555",
    "success": "#50fa7b"
  }
}
```

Plugin themes are read-only. Pressing `Ctrl+E` on one in `/theme` copies it into `~/.claude/themes/` so the user can edit the copy. Selecting a plugin theme persists `custom:<plugin-name>:<slug>` in the user's config.

## Related concepts

- [Plugin Manifest Schema](./plugin-manifest-schema.md) — component path fields and manifest configuration
- [Plugin Directory Structure](./plugin-directory-structure.md) — standard layout and file locations reference
- [Plugin Environment Variables](./plugin-environment-variables.md) — `CLAUDE_PLUGIN_ROOT`, `CLAUDE_PLUGIN_DATA`, `CLAUDE_PROJECT_DIR`
- [Hook Lifecycle Events](./hook-lifecycle-events.md) — the 30+ events plugins can hook
- [Hook Types](./hook-types.md) — command, http, mcp_tool, prompt, agent
