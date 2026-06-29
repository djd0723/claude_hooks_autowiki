---
type: concept
title: "Plugin Development Workflow"
created: 2026-06-29
updated: 2026-06-29
tags: [plugins, development, testing, plugin-dir, plugin-url, reload-plugins, debugging]
source_count: 1
sources:
  - sources/clean/plugins.md
---

# Plugin Development Workflow

While building a plugin you load it directly from disk — no install, no marketplace — and reload it in place as you edit. This page covers the local dev loop: loading, hot-reloading, testing components, and debugging.

## Load a plugin without installing

The `--plugin-dir` flag loads a plugin directly during development:

```bash
claude --plugin-dir ./my-plugin
```

The flag also accepts a `.zip` archive of the plugin directory, which requires Claude Code v2.1.128 or later:

```bash
claude --plugin-dir ./my-plugin.zip
```

Load multiple plugins by repeating the flag:

```bash
claude --plugin-dir ./plugin-one --plugin-dir ./plugin-two
```

### Local copy takes precedence

> When a `--plugin-dir` plugin has the same name as an installed marketplace plugin, the local copy takes precedence for that session. This lets you test changes to a plugin you already have installed without uninstalling it first.

The exception: plugins that managed settings force-enable or force-disable — `--plugin-dir` cannot override those.

## Load a hosted archive with `--plugin-url`

To test a plugin already packaged as a `.zip` and hosted at a URL (such as a CI build artifact), use `--plugin-url` instead. Claude Code fetches the archive at startup and loads it for that session only.

> If the fetch fails or the archive is invalid, Claude Code reports a plugin load error and starts without it.

The same [trust considerations](./plugin-distribution.md) apply as for any plugin source: only point this flag at archives you control or trust. Load multiple by repeating the flag, or pass space-separated URLs as one quoted argument:

```bash
claude --plugin-url "https://example.com/my-plugin.zip https://example.com/other.zip"
```

## Hot-reload with `/reload-plugins`

As you make changes, run `/reload-plugins` to pick up updates without restarting.

> This reloads plugins, skills, agents, hooks, plugin MCP servers, and plugin LSP servers.

Then test the components:

- Try your skills with `/plugin-name:skill-name`
- Check that agents appear in `/agents`
- Verify hooks work as expected

## Debug plugin issues

If your plugin isn't working as expected:

1. **Check the structure**: Ensure your directories are at the plugin root, not inside `.claude-plugin/`. This is the most common mistake — see [Creating Plugins](./creating-plugins.md#the-claude-plugin-boundary).
2. **Test components individually**: Check each skill, agent, and hook separately.
3. **Use validation and debugging tools**: run [`claude plugin validate`](./plugin-cli-commands.md#plugin-validate) and the other debugging CLI commands.

## Related concepts

- [Creating Plugins](./creating-plugins.md) — the authoring hub and minimal anatomy
- [Plugin CLI Commands](./plugin-cli-commands.md) — `validate`, `list`, `details` for inspecting a plugin
- [Plugin Distribution](./plugin-distribution.md) — moving from local dev to a marketplace
- [Skills-directory plugins](./plugin-installation-scopes.md#skills-directory-plugins) — auto-load from your skills directory instead of `--plugin-dir`
- [Plugin Versioning](./plugin-versioning.md) — mid-session reload behavior for hooks, monitors, and servers
