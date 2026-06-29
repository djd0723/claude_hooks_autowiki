---
type: comparison
title: "Plugin installation scopes vs hook configuration scopes"
created: 2026-06-29
updated: 2026-06-29
tags: [plugins, hooks, scopes, settings, configuration, managed]
sources:
  - sources/clean/code-claude-com-docs-en-plugins-reference-md.md
  - sources/clean/code-claude-com-docs-en-hooks-md.md
---

# Plugin installation scopes vs hook configuration scopes

Both plugins and hooks use a four-tier scope model backed by the same settings files. The scope
you choose determines availability, shareability, and how difficult it is to override. Despite the
shared file locations, the two systems differ in how scope restricts capability.

## Side-by-side scope table

| Scope | Settings file | Hook behavior | Plugin behavior |
| :---- | :------------ | :------------ | :-------------- |
| `user` | `~/.claude/settings.json` | Available in every project | Installed for every project (default) |
| `project` | `.claude/settings.json` | Shareable via version control | Shareable via version control |
| `local` | `.claude/settings.local.json` | Project-specific, gitignored | Project-specific, gitignored |
| `managed` | Managed policy settings | Admin-controlled; can't be disabled by users | Read-only; users can only update, not remove |

## Where they diverge

**Capability restrictions by scope.** Plugin scope carries a security dimension that hook scope does
not. Project-scope `@skills-dir` plugins load with restricted capabilities:

- MCP servers require per-server approval (same trust model as project `.mcp.json`)
- LSP servers start only after the workspace trust dialog is accepted
- Background monitors **do not load** at all

Hook scope has no equivalent restrictions — a project-scope hook fires with the same power as a
user-scope hook.

**Enterprise overrides.** On the hook side, administrators can set `allowManagedHooksOnly: true` to
block all user, project, and plugin hooks (vetted plugins in `enabledPlugins` are exempt). The plugin
side has no equivalent blanket block; managed scope makes plugins read-only but doesn't suppress
user/project plugins.

**`managed` mutability.** Managed hooks are simply admin-defined hooks that users cannot disable.
Managed plugins are also admin-controlled, but carry an additional constraint: users can run
`claude plugin update` but cannot uninstall them — the update-only restriction has no direct analogue
in hook configuration.

**Skills-dir loading.** A plugin discovered via the skills directory (`<name>@skills-dir`) inherits
scope from which directory it lives in (`~/.claude/skills/` → user scope; `.claude/skills/` →
project scope). The project-scope variant has the capability restrictions above. Plain hook
definitions in the same settings file do not carry these restrictions.

## Common pattern: plugin-shipped hooks

Because plugins can bundle a `hooks/hooks.json`, plugin scope and hook scope are often composed:
installing a plugin at project scope both registers the plugin and registers its hooks at project
scope. The hook-scope rules from the settings file layer on top of whatever scope the plugin itself
was installed at.

```json
// plugin.json (excerpt)
{
  "hooks": {
    "PostToolUse": [
      { "matcher": "Edit|Write", "hooks": [{ "type": "command", "command": "npx prettier --write \"$TOOL_ARG_FILE_PATH\"" }] }
    ]
  }
}
```

## When scope choice differs between the two systems

Use **user scope** for hooks and plugins you want everywhere — personal formatters, audit loggers.
Use **project scope** when the hook or plugin belongs to the repo, but be aware that project-scope
plugins face the trust restrictions above. Use **local scope** for experiment-or-machine-specific
overrides you don't want in version control.

## Related concepts

- [Plugin Installation Scopes](../concepts/plugin-installation-scopes.md) — full scope table and skills-directory details
- [Hook Scope and Configuration Location](../concepts/hook-scope.md) — hook scope table and managed-hooks enterprise rules
- [Plugin Components](../concepts/plugin-components.md) — how hooks are one of seven plugin component types
