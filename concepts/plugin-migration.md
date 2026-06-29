---
type: concept
title: "Plugin Migration"
created: 2026-06-29
updated: 2026-06-29
tags: [plugins, migration, standalone, conversion, claude-directory, hooks]
source_count: 1
sources:
  - sources/clean/code-claude-com-docs-en-plugins-md.md
---

# Plugin Migration

If you already have skills or hooks in your `.claude/` directory, you can convert them into a plugin for easier sharing and distribution. This is the graduation step from [standalone configuration](./plugins-vs-standalone.md).

## Migration steps

1. **Create the plugin structure** — a new directory with a `.claude-plugin/plugin.json` manifest:

   ```json
   {
     "name": "my-plugin",
     "description": "Migrated from standalone configuration",
     "version": "1.0.0"
   }
   ```

2. **Copy your existing files** — commands, agents, and skills move to the plugin root:

   ```bash
   cp -r .claude/commands my-plugin/
   cp -r .claude/agents my-plugin/
   cp -r .claude/skills my-plugin/
   ```

3. **Migrate hooks** — hooks move out of settings into a dedicated file:

   > Create `my-plugin/hooks/hooks.json` with your hooks configuration. Copy the `hooks` object from your `.claude/settings.json` or `settings.local.json`, since the format is the same. The command receives hook input as JSON on stdin, so use `jq` to extract the file path.

   ```json
   {
     "hooks": {
       "PostToolUse": [
         {
           "matcher": "Write|Edit",
           "hooks": [{ "type": "command", "command": "jq -r '.tool_input.file_path' | xargs npm run lint:fix" }]
         }
       ]
     }
   }
   ```

4. **Test your migrated plugin** — load it with `claude --plugin-dir ./my-plugin` and verify each component (see [Plugin Development Workflow](./plugin-development-workflow.md)).

## What changes when migrating

| Standalone (`.claude/`) | Plugin |
| :---------------------- | :----- |
| Only available in one project | Can be shared via marketplaces |
| Files in `.claude/commands/` | Files in `plugin-name/commands/` |
| Hooks in `settings.json` | Hooks in `hooks/hooks.json` |
| Must manually copy to share | Install with `/plugin install` |

## Remove the originals to avoid duplicates

> After migrating, remove the original files from `.claude/` to avoid duplicates. Project and user `.claude/agents/` definitions override same-named plugin agents, so the plugin version only takes effect once the originals are removed.

This override precedence is the same one that governs [skills-directory plugins](./plugin-installation-scopes.md#skills-directory-plugins): local `.claude/` definitions win over plugin-shipped ones.

## Related concepts

- [Plugins vs Standalone Configuration](./plugins-vs-standalone.md) — the decision this migration completes
- [Creating Plugins](./creating-plugins.md) — the authoring hub
- [Plugin Development Workflow](./plugin-development-workflow.md) — testing the migrated plugin
- [Hook Scope](./hook-scope.md) — how the migrated `hooks.json` is loaded as plugin-scope hooks
- [Plugin Distribution](./plugin-distribution.md) — sharing the plugin once migrated
