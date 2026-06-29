---
type: concept
title: "Plugin Installation Scopes"
created: 2026-06-29
updated: 2026-06-29
tags: [plugins, scopes, installation, settings, skills-directory, enabled-plugins, marketplaces]
source_count: 2
sources:
  - sources/clean/code-claude-com-docs-en-plugins-reference-md.md
  - sources/clean/code-claude-com-docs-en-settings-md.md
---

# Plugin Installation Scopes

When you install a plugin, you choose a **scope** that determines where it is available and who can use it.

## The four scopes

| Scope | Settings file | Use case |
| :---- | :------------ | :------- |
| `user` | `~/.claude/settings.json` | Personal plugins available across all projects (default) |
| `project` | `.claude/settings.json` | Team plugins shared via version control |
| `local` | `.claude/settings.local.json` | Project-specific plugins, gitignored |
| `managed` | Managed settings | Admin-controlled plugins (read-only, update only) |

The scope system is the same as other Claude Code configurations. Specify scope with `-s, --scope <scope>` on CLI commands; default is `user`.

## Skills-directory plugins

Any folder under a skills directory that contains a `.claude-plugin/plugin.json` manifest is loaded as `<name>@skills-dir` — no marketplace, no install step. Scaffold with `claude plugin init`.

| Skills directory | Scope | When it loads |
| :--------------- | :---- | :------------ |
| `~/.claude/skills/` | personal | Every project |
| `<cwd>/.claude/skills/` | project | Only after workspace trust dialog is accepted |

A skills-directory plugin differs from:
- A **plain skill** — just a `SKILL.md` file with no manifest; only provides a slash command
- A **marketplace-installed plugin** — copied to the plugin cache (`~/.claude/plugins/cache`); a skills-dir plugin is discovered in place

### Project-scope `@skills-dir` restrictions

Project-scope plugins load from `.claude/skills/` only when the workspace trust dialog is accepted. Because the content comes from the repository rather than from you personally:

- MCP servers need [per-server approval](../concepts/hook-scope.md) (same as project `.mcp.json`)
- LSP servers start only after workspace trust
- Background [monitors](./plugin-components.md#monitors) do **not** load

Personal-scope plugins (`~/.claude/skills/`) have none of these restrictions.

> **Warning**: Project-scope `@skills-dir` plugins load only from the `.claude/skills/` of the directory where you start Claude Code. They do not walk up to the repository root the way plain skills and commands do. Launch from the repository root, or run `/reload-plugins` after changing directories.

## Plugin caching for marketplace installs

Claude Code copies marketplace plugins into the user's local **plugin cache** (`~/.claude/plugins/cache`) rather than using them in-place. Each installed version is a separate directory. Previous versions are marked orphaned and removed automatically 7 days after an update (so concurrent sessions using the old version keep working). Claude's Glob and Grep tools skip orphaned version directories.

### Path traversal limitations

Installed plugins cannot reference files outside their directory. Paths that traverse outside the plugin root (e.g., `../shared-utils`) will not work after installation because those external files are not copied to the cache.

### Sharing files within a marketplace via symlinks

| Symlink target | Behavior in cache |
| :------------- | :---------------- |
| Within the plugin's own directory | Preserved as a relative symlink |
| Elsewhere within the same marketplace | Dereferenced — target content copied into cache |
| Outside the marketplace | Skipped for security |

For plugins installed with `--plugin-dir` or from a local path, only symlinks resolving within the plugin's own directory are preserved.

## Configuring plugins in `settings.json`

The scope a plugin lives in is just a `settings.json` at that scope. Two keys drive plugin configuration (see [Settings Files](./settings-files.md)):

### `enabledPlugins`

Maps `"plugin-name@marketplace-name"` to `true`/`false`. A plugin with no entry at any scope falls back to its [`defaultEnabled`](./plugin-manifest-schema.md) value.

```json
{
  "enabledPlugins": {
    "code-formatter@team-tools": true,
    "experimental-features@personal": false
  }
}
```

Precedence follows the [normal scope order](./settings-precedence.md): **project settings override user settings**, so setting a plugin to `false` in `~/.claude/settings.json` does *not* disable one the project's `.claude/settings.json` enables — opt out in `.claude/settings.local.json` instead. Plugins force-enabled by managed settings cannot be disabled at all.

### `extraKnownMarketplaces`

Declares additional marketplaces (typically in repository settings so teammates are prompted to install them on trust). Each entry's `source` selects a type:

| Source type | Uses |
| :---------- | :--- |
| `github` | GitHub repo (`repo`) |
| `git` | Any git URL (`url`) — works with self-hosted GitLab/Bitbucket |
| `directory` | Local filesystem path (`path`, development only) |
| `hostPattern` | Regex matching marketplace hosts |
| `settings` | Inline marketplace declared in `settings.json` (`name` + `plugins`); plugins must reference external sources |

Optional per-entry flags: `skipLfs` (skip Git LFS downloads, v2.1.153+) and `autoUpdate` (refresh and update on startup — defaults `true` for official Anthropic marketplaces, `false` otherwise).

### Managed marketplace restrictions

Managed settings can lock down marketplace additions with `strictKnownMarketplaces` and `blockedMarketplaces` — blocked sources are checked before download so they never touch the filesystem.

## Related concepts

- [Creating Plugins](./creating-plugins.md) — authoring a plugin, including the skills-directory dev path
- [Plugin Manifest Schema](./plugin-manifest-schema.md) — scope-related manifest fields (`defaultEnabled`)
- [Plugin CLI Commands](./plugin-cli-commands.md) — `install`, `uninstall`, `enable`, `disable` with `--scope` flags
- [Plugin Versioning](./plugin-versioning.md) — how the cache key determines available updates
- [Plugin Directory Structure](./plugin-directory-structure.md) — where components live within a plugin
