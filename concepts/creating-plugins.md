---
type: concept
title: "Creating Plugins"
created: 2026-06-29
updated: 2026-06-29
tags: [plugins, authoring, quickstart, manifest, skills, namespacing]
source_count: 1
sources:
  - sources/clean/plugins.md
---

# Creating Plugins

> Plugins let you extend Claude Code with custom functionality that can be shared across projects and teams.

A plugin is a self-contained directory bundling [skills, agents, hooks, MCP servers](./plugin-components.md), and more, optionally alongside a `.claude-plugin/plugin.json` manifest. This page is the hub for *authoring* a plugin — the decision to package one, the minimal anatomy, the dev loop, migration, and distribution. For the complete technical specifications, see [Plugin Directory Structure](./plugin-directory-structure.md) and [Plugin Manifest Schema](./plugin-manifest-schema.md).

## When to package a plugin

Claude Code supports two ways to add custom skills, agents, and hooks: **standalone** configuration in a `.claude/` directory, or a **plugin**. The dividing line is sharing and reuse — see [Plugins vs Standalone Configuration](./plugins-vs-standalone.md) for the full decision.

> Start with standalone configuration in `.claude/` for quick iteration, then [convert to a plugin](./plugin-migration.md) when you're ready to share.

## Minimal anatomy

The smallest useful plugin is a directory with a manifest and one skill:

```text
my-first-plugin/
├── .claude-plugin/
│   └── plugin.json
└── skills/
    └── hello/
        └── SKILL.md
```

The manifest at `.claude-plugin/plugin.json` defines the plugin's identity:

```json
{
  "name": "my-first-plugin",
  "description": "A greeting plugin to learn the basics",
  "version": "1.0.0",
  "author": {
    "name": "Your Name"
  }
}
```

The `name` field is both the unique identifier and the **skill namespace**: skills are prefixed with it, so `skills/hello/SKILL.md` becomes `/my-first-plugin:hello`. See [Plugin Manifest Schema](./plugin-manifest-schema.md) for every field.

Skills live in the `skills/` directory, each a folder containing a `SKILL.md` file:

> The folder name becomes the skill name, prefixed with the plugin's namespace (`hello/` in a plugin named `my-first-plugin` creates `/my-first-plugin:hello`).

A skill accepts user input through the `$ARGUMENTS` placeholder, which captures any text the user provides after the skill name. See [Skill Arguments](./skill-arguments.md) for the full substitution syntax.

## Why skills are always namespaced

> Plugin skills are always namespaced (like `/my-first-plugin:hello`) to prevent conflicts when multiple plugins have skills with the same name.

To change the namespace prefix, update the `name` field in `plugin.json`. This is the key behavioral difference from standalone skills, which use short unprefixed names like `/hello`.

## The `.claude-plugin/` boundary

A common mistake when laying out a plugin:

> Don't put `commands/`, `agents/`, `skills/`, or `hooks/` inside the `.claude-plugin/` directory. Only `plugin.json` goes inside `.claude-plugin/`. All other directories must be at the plugin root level.

This rule is the first thing to check when [debugging a plugin](./plugin-development-workflow.md#debug-plugin-issues) that isn't loading.

## Single-skill shortcut

> A plugin that ships exactly one skill can place `SKILL.md` directly at the plugin root instead of creating a `skills/` directory.

Claude Code loads it as a single skill and uses the frontmatter `name` field for the invocation name. Use the `skills/` layout for plugins that may grow to more than one skill.

## Related concepts

- [Plugins vs Standalone Configuration](./plugins-vs-standalone.md) — when to package a plugin vs use `.claude/`
- [Plugin Development Workflow](./plugin-development-workflow.md) — `--plugin-dir`, `/reload-plugins`, testing, debugging
- [Plugin Migration](./plugin-migration.md) — converting standalone `.claude/` configs into a plugin
- [Plugin Distribution](./plugin-distribution.md) — marketplaces, community submission, validation
- [Plugin Default Settings](./plugin-default-settings.md) — shipping a `settings.json` that activates an agent
- [Skills-directory plugins](./plugin-installation-scopes.md#skills-directory-plugins) — develop a plugin in your skills directory with no install step
- [Plugin Components](./plugin-components.md) — the seven component types a plugin can ship
- [Plugin Manifest Schema](./plugin-manifest-schema.md) — every `plugin.json` field
