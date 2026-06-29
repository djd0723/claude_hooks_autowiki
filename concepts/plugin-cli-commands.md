---
type: concept
title: "Plugin CLI Commands"
created: 2026-06-29
updated: 2026-06-29
tags: [plugins, cli, commands, install, uninstall, enable, disable, validate]
source_count: 1
sources:
  - sources/clean/code-claude-com-docs-en-plugins-reference-md.md
---

# Plugin CLI Commands

Claude Code provides CLI commands for non-interactive plugin management, useful for scripting and automation. All plugin commands follow the form `claude plugin <subcommand>`.

## `plugin init` (alias: `new`)

Scaffold a new plugin at `~/.claude/skills/<name>/`. On the next Claude Code session it loads automatically as `<name>@skills-dir`.

```bash
claude plugin init <name> [options]
```

| Option | Description | Default |
| :----- | :---------- | :------ |
| `--description <text>` | Manifest description | |
| `--author <name>` | Author name | `git config user.name` |
| `--author-email <email>` | Author email | `git config user.email` |
| `--with <components...>` | Also scaffold component folders | |
| `-f, --force` | Overwrite an existing `.claude-plugin/` at the target | |

Valid `--with` values: `skills`, `agents`, `hooks`, `mcp`, `lsp`, `output-style`, `channel`. Each adds a starter file ready to edit.

## `plugin install`

Install a plugin from available marketplaces.

```bash
claude plugin install <plugin> [options]
# e.g. claude plugin install formatter@my-marketplace --scope project
```

| Option | Description | Default |
| :----- | :---------- | :------ |
| `-s, --scope <scope>` | Installation scope: `user`, `project`, or `local` | `user` |

Scope determines which settings file the plugin is added to. `--scope project` writes to `.claude/settings.json`, making the plugin available to everyone who clones the repository.

## `plugin uninstall` (aliases: `remove`, `rm`)

Remove an installed plugin.

```bash
claude plugin uninstall <plugin> [options]
```

| Option | Description |
| :----- | :---------- |
| `-s, --scope <scope>` | Uninstall from: `user`, `project`, or `local` (default: `user`) |
| `--keep-data` | Preserve the plugin's persistent data directory |
| `--prune` | Also remove auto-installed dependencies no other plugin requires |
| `-y, --yes` | Skip the `--prune` confirmation prompt |

By default, uninstalling from the last scope also deletes `${CLAUDE_PLUGIN_DATA}`. Use `--keep-data` to preserve it (e.g., when reinstalling to test a new version).

## `plugin prune` (alias: `autoremove`)

Remove auto-installed plugin dependencies that are no longer required by any installed plugin. Plugins you installed directly are never touched. Requires Claude Code v2.1.121+.

```bash
claude plugin prune [options]
```

| Option | Description |
| :----- | :---------- |
| `-s, --scope <scope>` | Prune at: `user`, `project`, or `local` (default: `user`) |
| `--dry-run` | List what would be removed without removing anything |
| `-y, --yes` | Skip confirmation prompt |

To remove a plugin and clean up its dependencies in one step: `claude plugin uninstall <plugin> --prune`.

## `plugin enable`

Enable a disabled plugin. If the plugin declares dependencies, Claude Code enables them transitively at the same scope. Fails when a dependency is not installed.

```bash
claude plugin enable <plugin> [-s, --scope <scope>]
```

## `plugin disable`

Disable a plugin without uninstalling it. Fails when another enabled plugin depends on the target — the error message includes a chained command that disables every dependent first.

```bash
claude plugin disable <plugin> [-s, --scope <scope>]
```

## `plugin update`

Update a plugin to the latest version.

```bash
claude plugin update <plugin> [-s, --scope <scope>]
```

Scope can also be `managed` for managed plugins. See [Plugin Versioning](./plugin-versioning.md) for when an update is detected.

## `plugin list`

List installed plugins with their version, source marketplace, and enable status.

```bash
claude plugin list [--json] [--available] [-h]
```

`--available` requires `--json`. Lists available plugins from marketplaces alongside installed ones.

Within an interactive session, `/plugin list` (or `/plugin ls`) prints the same listing inline. The interactive form accepts `--enabled` or `--disabled` to filter.

## `plugin details`

Show a plugin's component inventory and projected token cost.

```bash
claude plugin details <name>
```

Output includes:
- **Component inventory**: Skills, Agents, Hooks, MCP servers, LSP servers
- **Always-on cost**: tokens added to every session (from listing text like skill/agent descriptions)
- **Per-component on-invoke cost**: tokens paid each time a skill or agent fires

Token counts are estimates via the `count_tokens` API; falls back to character-based estimates if the API is unreachable.

## `plugin validate`

Check a plugin's manifest, skill/agent frontmatter, and hooks.json for syntax and schema errors.

```bash
claude plugin validate ./my-plugin [--strict]
```

`--strict` treats unrecognized-field warnings as errors. Use in CI to catch misspelled field names before publishing. Without `--strict`, plugins with only unrecognized-field warnings still pass validation and load at runtime.

Fields with the wrong type (e.g., `keywords` as a string instead of an array) always fail as errors regardless of `--strict`.

## `plugin tag`

Create a release git tag for the plugin in the current directory. Run from inside the plugin's folder.

```bash
claude plugin tag [--push] [--dry-run] [-f, --force]
```

| Option | Description |
| :----- | :---------- |
| `--push` | Push the tag to the remote after creating it |
| `--dry-run` | Print what would be tagged without creating the tag |
| `-f, --force` | Create the tag even if the working tree is dirty or the tag already exists |

## Related concepts

- [Plugin Installation Scopes](./plugin-installation-scopes.md) — user/project/local/managed scope details
- [Plugin Versioning](./plugin-versioning.md) — how `plugin update` detects new versions
- [Plugin Environment Variables](./plugin-environment-variables.md) — `CLAUDE_PLUGIN_DATA` and `--keep-data`
- [Plugin Directory Structure](./plugin-directory-structure.md) — what `plugin init` scaffolds
