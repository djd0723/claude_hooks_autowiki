---
type: concept
title: "Plugin Default Settings"
created: 2026-06-29
updated: 2026-06-29
tags: [plugins, settings, agent, subagentStatusLine, defaults]
source_count: 1
sources:
  - sources/clean/code-claude-com-docs-en-plugins-md.md
---

# Plugin Default Settings

A plugin can ship a `settings.json` file at its root to apply default configuration when the plugin is enabled. This lets a plugin change how Claude Code behaves by default — most powerfully, by activating one of its own agents as the main thread.

## The `settings.json` file

> Plugins can include a `settings.json` file at the plugin root to apply default configuration when the plugin is enabled. Currently, only the `agent` and `subagentStatusLine` keys are supported.

```json
{
  "agent": "security-reviewer"
}
```

## Activating an agent as the main thread

> Setting `agent` activates one of the plugin's custom agents as the main thread, applying its system prompt, tool restrictions, and model. This lets a plugin change how Claude Code behaves by default when enabled.

The example above activates the `security-reviewer` agent defined in the plugin's `agents/` directory. See [Plugin Components](./plugin-components.md#agents) for how plugin-shipped agents are defined and [Subagents](./subagents.md) for agent configuration in general.

## Precedence and unknown keys

Two rules govern how this file is merged:

- **Settings from `settings.json` take priority over `settings` declared in `plugin.json`.**
- **Unknown keys are silently ignored.**

The first rule means the root `settings.json` is the authoritative override; the inline `settings` block in [the manifest](./plugin-manifest-schema.md) is the lower-priority default.

## Related concepts

- [Plugin Components](./plugin-components.md) — the `agents/` directory whose agents `settings.json` can activate
- [Subagents](./subagents.md) — agent system prompts, tool restrictions, and model selection
- [Plugin Manifest Schema](./plugin-manifest-schema.md) — the lower-priority inline `settings` block
- [Creating Plugins](./creating-plugins.md) — the authoring hub
