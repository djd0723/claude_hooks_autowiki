---
type: comparison
title: "Skills-directory plugins vs marketplace-installed plugins"
created: 2026-06-29
updated: 2026-06-29
tags: [plugins, skills-dir, marketplace, installation, cache, scopes]
sources:
  - sources/clean/code-claude-com-docs-en-plugins-reference-md.md
---

# Skills-directory plugins vs marketplace-installed plugins

Claude Code supports two discovery mechanisms for plugins. A **skills-directory plugin** is found
in-place inside a `skills/` folder; a **marketplace-installed plugin** is fetched and copied to the
plugin cache. The mechanism determines update semantics, path constraints, and which capabilities
are available.

## Side-by-side comparison

| Dimension | Skills-directory plugin | Marketplace-installed plugin |
| :-------- | :---------------------- | :--------------------------- |
| Discovery | Detected automatically inside `~/.claude/skills/` or `.claude/skills/` | Installed via `claude plugin install` |
| File location | Lives in-place at the skills directory; never moved | Copied to `~/.claude/plugins/cache/<name>/<version>/` |
| Identifier | `<name>@skills-dir` | `<name>` or `<name>@<source>` |
| Scaffold command | `claude plugin init` inside the target directory | N/A — install from a source |
| Updates | Modified in place; reload with `/reload-plugins` | `claude plugin update`; previous versions marked orphaned for 7 days |
| Path traversal | Can reference files anywhere on disk (in-place) | Cannot traverse outside the plugin directory; paths like `../shared-utils` break after installation |
| Symlink handling | Preserved if within the plugin directory | Symlinks within the plugin: kept as relative symlinks; symlinks to other plugins in the same marketplace: dereferenced (target content copied in); symlinks outside the marketplace: skipped for security |
| Project-scope restrictions | MCP servers need per-server approval; LSP servers need workspace trust dialog; background monitors **do not load** | No equivalent restrictions — all components load normally |
| Working-directory sensitivity | Must launch Claude Code from the directory where the plugin lives; does not walk up to repo root | No working-directory sensitivity |

## When to use each

**Skills-directory plugin**: the right choice when you are actively developing a plugin, want zero
install friction for team members who clone a repo, or need to bundle components for a project
without marketplace overhead. The trade-off is that the developer bears responsibility for keeping
the directory clean and reloading manually after edits.

**Marketplace-installed plugin**: the right choice for distributing a finished plugin to users who
should not need to manage the source directory. The cache model isolates versions, allows concurrent
sessions to keep using an old version while the new one is staged, and automatically prunes stale
versions after seven days.

## Manifest requirement

Both mechanisms discover a plugin by its `.claude-plugin/plugin.json` manifest (or fall back to the
directory name when no manifest is present). For marketplace plugins, always set a stable `name` in
the manifest — the cache directory name becomes a version string on updates, so without an explicit
`name`, Claude Code cannot reliably identify the plugin across versions.

> "If a manifest is present, `name` is the only required field."

## Pruning and orphans (marketplace only)

When you run `claude plugin update`, the previous version is marked **orphaned**. It stays in the
cache for seven days so concurrent sessions using it keep working, then is removed automatically by
`claude plugin prune` (which you can run manually to reclaim disk immediately). Claude's Glob and
Grep tools skip orphaned version directories.

## Related concepts

- [Plugin Installation Scopes](../concepts/plugin-installation-scopes.md) — scope table and project-scope restrictions for `@skills-dir`
- [Plugin Versioning](../concepts/plugin-versioning.md) — version resolution order and update semantics
- [Plugin Directory Structure](../concepts/plugin-directory-structure.md) — standard layout for both plugin types
- [Plugin CLI Commands](../concepts/plugin-cli-commands.md) — `install`, `update`, `prune`, `init` commands
