---
type: concept
title: "Plugin Versioning"
created: 2026-06-29
updated: 2026-06-29
tags: [plugins, versioning, version-management, cache, updates]
source_count: 1
sources:
  - sources/clean/code-claude-com-docs-en-plugins-reference-md.md
---

# Plugin Versioning

Claude Code uses a plugin's version as the cache key that determines whether an update is available. When `/plugin update` or auto-update fires, Claude Code computes the current version and skips the update if it matches what's already installed.

## Version resolution order

The version is resolved from the first of these that is set:

1. `version` field in `plugin.json`
2. `version` field in the plugin's marketplace entry (`marketplace.json`)
3. Git commit SHA of the plugin's source (for `github`, `url`, `git-subdir`, and relative-path sources in a git-hosted marketplace)
4. `"unknown"` (for `npm` sources or local directories not inside a git repository)

If `version` is set in `plugin.json`, it wins over the marketplace entry.

## Two versioning strategies

| Strategy | How | Update behavior | Best for |
| :-------- | :-- | :-------------- | :------- |
| **Explicit version** | Set `"version": "2.1.0"` in `plugin.json` | Users get updates only when you bump this field. Pushing new commits without bumping has no effect. | Published plugins with stable release cycles |
| **Commit-SHA version** | Omit `version` from both `plugin.json` and the marketplace entry | Every new commit is treated as a new version | Internal or team plugins under active development |

> **Warning**: If you set `version` in `plugin.json`, you must bump it every time you want users to receive changes. Pushing new commits alone is not enough — Claude Code sees the same version string and keeps the cached copy. If iterating quickly, leave `version` unset so the git commit SHA is used instead.

## Semantic versioning

When using explicit versions, follow [semantic versioning](https://semver.org) (`MAJOR.MINOR.PATCH`):
- **MAJOR**: breaking changes
- **MINOR**: new features
- **PATCH**: bug fixes

Document changes in a `CHANGELOG.md`.

## Update mechanics

- Each installed version is a **separate directory** in the plugin cache (`~/.claude/plugins/cache`)
- When you update or uninstall a plugin, the previous version directory is marked as **orphaned** and removed automatically 7 days later
- The 7-day grace period lets concurrent Claude Code sessions that already loaded the old version keep running without errors
- Claude's Glob and Grep tools skip orphaned version directories during searches

## Mid-session update behavior

When a plugin updates while a session is running:
- **Hook commands, monitors, MCP servers, and LSP servers** keep using the previous version's path
- Run `/reload-plugins` to switch hooks, MCP servers, and LSP servers to the new path
- **Monitors** require a full session restart to pick up the new path

## `plugin tag` command

Use `claude plugin tag` to create a release git tag from inside the plugin's folder. See [Plugin CLI Commands](./plugin-cli-commands.md#plugin-tag) for options.

## Related concepts

- [Plugin Manifest Schema](./plugin-manifest-schema.md) — `version` and `defaultEnabled` fields
- [Plugin CLI Commands](./plugin-cli-commands.md) — `update`, `uninstall --keep-data`, `tag`
- [Plugin Installation Scopes](./plugin-installation-scopes.md) — cache location and orphan cleanup
- [Plugin Environment Variables](./plugin-environment-variables.md) — `CLAUDE_PLUGIN_ROOT` path changes on update
