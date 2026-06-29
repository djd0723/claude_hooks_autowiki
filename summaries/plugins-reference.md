---
type: summary
title: "Plugins Reference — Technical specification for the Claude Code plugin system"
slug: code-claude-com-docs-en-plugins-reference-md
created: 2026-06-29
updated: 2026-06-29
tags: [plugins, reference, schema, cli, components, skills, agents, mcp, lsp, monitors]
sources:
  - sources/clean/code-claude-com-docs-en-plugins-reference-md.md
---

# Summary: Plugins Reference

Complete technical reference for the Claude Code plugin system. A **plugin** is a self-contained directory of components that extends Claude Code with custom functionality.

## What this source covers

- All seven [plugin component types](../concepts/plugin-components.md): skills, agents, hooks, MCP servers, LSP servers, monitors, themes
- [Plugin manifest schema](../concepts/plugin-manifest-schema.md) (`plugin.json`): complete field reference
- [Installation scopes](../concepts/plugin-installation-scopes.md): user, project, local, managed
- [Directory structure](../concepts/plugin-directory-structure.md): standard layout and file locations reference
- [Environment variables](../concepts/plugin-environment-variables.md): `CLAUDE_PLUGIN_ROOT`, `CLAUDE_PLUGIN_DATA`, `CLAUDE_PROJECT_DIR`
- Plugin caching, symlink resolution, and path traversal security rules
- [CLI commands](../concepts/plugin-cli-commands.md): init, install, uninstall, prune, enable, disable, update, list, details, tag
- [Version management](../concepts/plugin-versioning.md): explicit vs commit-SHA versioning
- Debugging commands and common error messages

## Core concept

> "A plugin is a self-contained directory of components that extends Claude Code with custom functionality. Plugin components include skills, agents, hooks, MCP servers, LSP servers, and monitors."

The manifest (`.claude-plugin/plugin.json`) is optional. Claude Code auto-discovers components in default locations and derives the plugin name from the directory name when no manifest exists.

## Component summary

| Component | Default location | Purpose |
| :-------- | :--------------- | :------ |
| Skills | `skills/` | `/name` shortcuts Claude or users can invoke |
| Agents | `agents/` | Specialized subagents for specific tasks |
| Hooks | `hooks/hooks.json` | Event handlers for lifecycle automation |
| MCP servers | `.mcp.json` | External tools and services via MCP |
| LSP servers | `.lsp.json` | Code intelligence (diagnostics, go-to-definition) |
| Monitors | `monitors/monitors.json` | Background watchers delivering stdout as notifications |
| Themes | `themes/` | Color themes for the `/theme` interface |

## Manifest: only `name` is required

If a manifest is present, `name` is the only required field. Without it, Claude Code uses the directory name as a fallback — but the directory name changes on marketplace updates (it becomes a version string), so always set a stable `name` in frontmatter.

Claude Code ignores unrecognized top-level manifest fields. This makes it practical to maintain one manifest that doubles as a VS Code extension manifest, `package.json`, or MCPB/DXT bundle. `claude plugin validate --strict` promotes unrecognized-field warnings to errors for CI.

## Scopes

| Scope | Settings file | Use case |
| :---- | :------------ | :------- |
| `user` | `~/.claude/settings.json` | Personal plugins across all projects (default) |
| `project` | `.claude/settings.json` | Team plugins in version control |
| `local` | `.claude/settings.local.json` | Project-specific, gitignored |
| `managed` | Managed settings | Read-only, admin-controlled; update only |

## Skills-directory plugins

Any folder under a skills directory with a `.claude-plugin/plugin.json` is loaded as `<name>@skills-dir`. No marketplace, no install step — scaffold with `claude plugin init`. Differs from a plain skill (just a `SKILL.md`) in that it can bundle hooks, agents, MCP servers, and more.

Project-scope `@skills-dir` plugins are restricted: MCP servers need per-server approval, LSP servers need workspace trust, and background monitors do not load.

## Version management

Version resolution order (first set wins):
1. `version` in `plugin.json`
2. `version` in marketplace entry
3. Git commit SHA (for git-hosted sources)
4. `"unknown"` (npm or non-git local)

If you set `version` explicitly, you must bump it for users to receive updates — pushing new commits alone has no effect. If left unset, every commit is treated as a new version.

## Debugging

- `claude --debug` shows plugin loading, manifest errors, and component registration
- `claude plugin validate ./my-plugin` — checks plugin.json, skill/agent frontmatter, and hooks.json
- `claude plugin validate --strict` — treats unrecognized-field warnings as errors (use in CI)
- `claude plugin details <name>` — shows component inventory and projected token cost
- `/plugin` interface: Discover tab, Errors tab, and per-plugin detail view
