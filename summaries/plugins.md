---
type: summary
title: "Create plugins — authoring guide and workflow"
slug: code-claude-com-docs-en-plugins-md
created: 2026-06-29
updated: 2026-06-29
tags: [plugins, authoring, workflow, distribution, marketplace, migration]
sources:
  - sources/clean/code-claude-com-docs-en-plugins-md.md
---

# Summary: Create plugins (authoring guide)

The how-to guide for building, testing, and distributing your own Claude Code plugins. It walks through scaffolding a manifest, adding components, local testing, and shipping to a marketplace. Companion to the [plugins reference summary](plugins-reference.md), which is the field-level schema reference — this guide covers the workflow, that one covers the specifications, so they do not duplicate each other.

## Key thesis from source

> "Plugins let you extend Claude Code with custom functionality that can be shared across projects and teams. This guide covers creating your own plugins with skills, agents, hooks, and MCP servers."

## What a plugin is, at a high level

A plugin is a self-contained directory holding skills, agents, hooks, and/or MCP servers, optionally alongside a `.claude-plugin/plugin.json` manifest. Once distributed, a plugin gives teams versioned, reusable, namespaced extensions instead of per-project copy-paste.

## When to use plugins vs standalone configuration

> "Claude Code supports two ways to add custom skills, agents, and hooks"

| Approach | Skill names | Best for |
| :------- | :---------- | :------- |
| **Standalone** (`.claude/` directory) | `/hello` | Personal workflows, project-specific customizations, quick experiments |
| **Plugins** (self-contained directories with a `.claude-plugin/plugin.json` manifest) | `/plugin-name:hello` | Sharing with teammates, distributing to community, versioned releases, reusable across projects |

The recommended path is to start standalone in `.claude/` for fast iteration, then [convert to a plugin](#convert-existing-configurations-to-plugins) when ready to share. See [plugins vs standalone](../concepts/plugins-vs-standalone.md).

## Authoring workflow

The guide follows an init → develop → test → distribute arc. See [plugin development workflow](../concepts/plugin-development-workflow.md).

### 1. Init — scaffold a manifest

The manifest at `.claude-plugin/plugin.json` defines the plugin's identity. The quickstart fields:

| Field | Purpose |
| :---- | :------ |
| `name` | Unique identifier and skill namespace; skills are prefixed (e.g., `/my-first-plugin:hello`). |
| `description` | Shown in the plugin manager when browsing or installing. |
| `version` | Optional. If set, users receive updates only when you bump it. If omitted and distributed via git, the commit SHA is used and every commit counts as a new version. |
| `author` | Optional. Helpful for attribution. |

For full field details (`homepage`, `repository`, `license`, version resolution order), see the [plugins reference summary](plugins-reference.md).

A faster on-ramp for a single-machine plugin: `claude plugin init my-tool` scaffolds `~/.claude/skills/my-tool/` with a manifest and starter `SKILL.md`. It auto-loads next session as `my-tool@skills-dir` — no marketplace or install step.

### 2. Develop — add components

> "you've created a plugin with a skill, but plugins can include much more: custom agents, hooks, MCP servers, LSP servers, and background monitors."

Skills live in `skills/<name>/SKILL.md`; the folder name becomes the namespaced skill name. The `$ARGUMENTS` placeholder captures user input after the skill name for dynamic behavior. Other components are added by dropping the right file at the plugin root: an `.lsp.json` for code intelligence, a `monitors/monitors.json` for background watchers, and a root `settings.json` to ship default config (only `agent` and `subagentStatusLine` keys are supported). See [plugin components](../concepts/plugin-components.md).

### 3. Test — local loading

| Flag / command | Effect |
| :------------- | :----- |
| `claude --plugin-dir ./my-plugin` | Loads a plugin directory (or a `.zip`, v2.1.128+) without installing; local copy takes precedence over a same-named installed plugin. |
| `claude --plugin-url https://…/my-plugin.zip` | Fetches and loads a hosted `.zip` for that session only. |
| `/reload-plugins` | Picks up edits without restarting (reloads plugins, skills, agents, hooks, plugin MCP and LSP servers). |

Repeat `--plugin-dir`/`--plugin-url` to load multiple plugins. Debug by checking the structure, testing components individually, and using validation tools.

### 4. Distribute — ship it

When ready to share: add a `README.md`, choose a versioning strategy (explicit `version` vs git SHA), create or use a [marketplace](../concepts/plugin-distribution.md), and have teammates test first. Keep a plugin internal by hosting its marketplace in a private repository.

## Directory layout

> "Common mistake: Don't put `commands/`, `agents/`, `skills/`, or `hooks/` inside the `.claude-plugin/` directory. Only `plugin.json` goes inside `.claude-plugin/`. All other directories must be at the plugin root level."

| Directory / file | Location | Purpose |
| :--------------- | :------- | :------ |
| `.claude-plugin/` | Plugin root | Contains `plugin.json` (optional if components use default locations) |
| `skills/` | Plugin root | Skills as `<name>/SKILL.md` directories |
| `commands/` | Plugin root | Skills as flat Markdown files (use `skills/` for new plugins) |
| `agents/` | Plugin root | Custom agent definitions |
| `hooks/` | Plugin root | Event handlers in `hooks.json` |
| `.mcp.json` | Plugin root | MCP server configurations |
| `.lsp.json` | Plugin root | LSP server configurations |
| `monitors/` | Plugin root | Background monitors in `monitors.json` |
| `bin/` | Plugin root | Executables added to the Bash tool's `PATH` while enabled |
| `settings.json` | Plugin root | Default settings applied when the plugin is enabled |

A plugin shipping exactly one skill can place `SKILL.md` directly at the root; use the `skills/` layout for anything that may grow. See [plugin directory structure](../concepts/plugin-directory-structure.md).

## Distribution and marketplaces

Anthropic maintains two public marketplaces:

| Marketplace | What it is | How users add it |
| :---------- | :--------- | :--------------- |
| `claude-plugins-official` | Curated set maintained by Anthropic; registered automatically on first interactive launch | `claude plugin marketplace add anthropics/claude-plugins-official` (only needed for non-interactive scripts) |
| `claude-community` | Public community marketplace for reviewed third-party submissions | `/plugin marketplace add anthropics/claude-plugins-community`, install as `@claude-community` |

Submit to the community marketplace via the in-app forms (claude.ai admin settings or the Console). Run `claude plugin validate` locally first — the review pipeline runs the same check plus automated safety screening. Approved plugins are pinned to a commit SHA in the community catalog, with CI bumping the pin and a nightly sync. The official marketplace is curated separately at Anthropic's discretion; there is no application process.

## Converting standalone configs to plugins

Migration steps: create `my-plugin/.claude-plugin/plugin.json`, copy existing `commands/`, `agents/`, and `skills/` into the plugin directory, and move the `hooks` object from `.claude/settings.json` into `my-plugin/hooks/hooks.json` (same format). What changes:

| Standalone (`.claude/`) | Plugin |
| :---------------------- | :----- |
| Only available in one project | Can be shared via marketplaces |
| Files in `.claude/commands/` | Files in `plugin-name/commands/` |
| Hooks in `settings.json` | Hooks in `hooks/hooks.json` |
| Must manually copy to share | Install with `/plugin install` |

After migrating, remove the originals from `.claude/` to avoid duplicates — project and user `.claude/agents/` definitions override same-named plugin agents, so the plugin version takes effect only once the originals are gone. See [plugin migration](../concepts/plugin-migration.md).

## Relationship to concept pages

This guide is the primary source for:
- [Creating plugins](../concepts/creating-plugins.md) — the overall authoring story
- [Plugin development workflow](../concepts/plugin-development-workflow.md) — init → develop → test → distribute
- [Plugin components](../concepts/plugin-components.md) — what you can add (skills, agents, hooks, MCP, LSP, monitors)
- [Plugin directory structure](../concepts/plugin-directory-structure.md) — root-level layout and the `.claude-plugin/` mistake
- [Plugins vs standalone](../concepts/plugins-vs-standalone.md) — when each approach fits
- [Plugin distribution](../concepts/plugin-distribution.md) — marketplaces and community submission
- [Plugin migration](../concepts/plugin-migration.md) — converting `.claude/` configs into a plugin
- [Plugin installation scopes](../concepts/plugin-installation-scopes.md) — where installed plugins live

For field schemas and CLI details rather than workflow, see the [plugins reference summary](plugins-reference.md).
