---
type: concept
title: "Configuration Scopes"
created: 2026-06-29
updated: 2026-06-29
tags: [settings, scopes, configuration, managed, enterprise]
source_count: 1
sources:
  - sources/clean/code-claude-com-docs-en-settings-md.md
---

# Configuration Scopes

Claude Code uses a **scope system** to decide where a configuration applies and who it is shared with. Understanding scopes is the entry point to configuring Claude Code for personal use, team collaboration, or enterprise deployment.

## The four scopes

| Scope | Location | Who it affects | Shared with team? |
| :---- | :------- | :------------- | :---------------- |
| **Managed** | Server-managed settings, plist / registry, or system-level `managed-settings.json` | All org members (server delivery); all machine users (plist, HKLM registry, file delivery); the current user (HKCU registry delivery) | Yes (deployed by IT) |
| **User** | `~/.claude/` directory | You, across all projects | No |
| **Project** | `.claude/` in repository | All collaborators on this repository | Yes (committed to git) |
| **Local** | `.claude/settings.local.json` | You, in this repository only | No (gitignored when Claude Code creates it) |

On Windows, paths shown as `~/.claude` resolve to `%USERPROFILE%\.claude`.

## When to use each scope

- **Managed** — security policies enforced org-wide, compliance requirements that can't be overridden, standardized IT/DevOps deployments.
- **User** — personal preferences you want everywhere (themes, editor settings), tools/plugins used across all projects, API keys and authentication.
- **Project** — team-shared settings (permissions, hooks, MCP servers), plugins the whole team should have, standardized tooling.
- **Local** — personal overrides for one project, testing before sharing, machine-specific settings.

## What uses scopes

Scopes apply to many features, each with its own file locations:

| Feature | User location | Project location | Local location |
| :------ | :------------ | :--------------- | :------------- |
| **Settings** | `~/.claude/settings.json` | `.claude/settings.json` | `.claude/settings.local.json` |
| **Subagents** | `~/.claude/agents/` | `.claude/agents/` | None |
| **MCP servers** | `~/.claude.json` | `.mcp.json` | `~/.claude.json` (per-project) |
| **Plugins** | `~/.claude/settings.json` | `.claude/settings.json` | `.claude/settings.local.json` |
| **CLAUDE.md** | `~/.claude/CLAUDE.md` | `CLAUDE.md` or `.claude/CLAUDE.md` | `CLAUDE.local.md` |

## How scopes interact

When the same setting appears in multiple scopes, Claude Code resolves the conflict by precedence — Managed wins, User loses. The full ordering and the special case where **arrays merge instead of override** live in [Settings Precedence](./settings-precedence.md).

> For example, if your user settings set `spinnerTipsEnabled` to `true` and project settings set it to `false`, the project value applies.

Permission rules are the notable exception: they **merge across scopes** rather than override — see [Permission Settings](./permission-settings.md).

## Related concepts

- [Settings Files](./settings-files.md) — the `settings.json` files and managed delivery mechanisms that back each scope
- [Settings Precedence](./settings-precedence.md) — the exact override order and array-merge rule
- [Permission Settings](./permission-settings.md) — permission rules merge across scopes
- [Plugin Installation Scopes](./plugin-installation-scopes.md) — the same four scopes applied to plugins
