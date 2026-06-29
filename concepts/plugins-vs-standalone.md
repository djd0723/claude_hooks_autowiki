---
type: concept
title: "Plugins vs Standalone Configuration"
created: 2026-06-29
updated: 2026-06-29
tags: [plugins, standalone, configuration, namespacing, decision]
source_count: 1
sources:
  - sources/clean/code-claude-com-docs-en-plugins-md.md
---

# Plugins vs Standalone Configuration

Claude Code supports two ways to add custom skills, agents, and hooks. The choice is fundamentally about **sharing**: a personal customization stays standalone; anything you want teammates or the community to reuse becomes a plugin.

## The two approaches

| Approach | Skill names | Best for |
| :------- | :---------- | :------- |
| **Standalone** (`.claude/` directory) | `/hello` | Personal workflows, project-specific customizations, quick experiments |
| **Plugins** (self-contained directories with skills, agents, hooks, or a `.claude-plugin/plugin.json` manifest) | `/plugin-name:hello` | Sharing with teammates, distributing to community, versioned releases, reusable across projects |

The most visible difference is the skill name. Standalone skills get short names like `/hello` or `/deploy`; plugin skills are always namespaced as `/plugin-name:hello`. Namespacing prevents conflicts between plugins.

## Use standalone configuration when

- You're customizing Claude Code for a single project
- The configuration is personal and doesn't need to be shared
- You're experimenting with skills or hooks before packaging them
- You want short skill names like `/hello` or `/deploy`

## Use plugins when

- You want to share functionality with your team or community
- You need the same skills/agents across multiple projects
- You want version control and easy updates for your extensions
- You're distributing through a marketplace
- You're okay with namespaced skills like `/my-plugin:hello`

## The recommended path

These aren't mutually exclusive over a project's life. The guidance is to start standalone and graduate:

> Start with standalone configuration in `.claude/` for quick iteration, then convert to a plugin when you're ready to share.

When that moment comes, see [Plugin Migration](./plugin-migration.md) for the mechanical steps and the table of exactly what changes.

## Related concepts

- [Creating Plugins](./creating-plugins.md) — the authoring hub
- [Plugin Migration](./plugin-migration.md) — converting a `.claude/` config into a plugin
- [Plugin Installation Scopes](./plugin-installation-scopes.md) — where an installed plugin lives (user/project/local/managed)
- [Skills](./skills.md) — how skills work in both standalone and plugin form
