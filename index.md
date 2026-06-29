---
type: index
created: 2026-06-29
updated: 2026-06-29
tags: [hooks, plugins, navigation, index, reference]
---

# Claude Code Hooks & Plugins

Master navigation. Maintained by the indexer. Start with [overview](overview.md)
and the running [synthesis](synthesis.md).

## Sources

- [Hooks Guide](sources/clean/code-claude-com-docs-en-hooks-guide-md.md) — official Claude Code hooks tutorial (clean source)
- [Hooks Reference](sources/clean/code-claude-com-docs-en-hooks-md.md) — official hooks reference: event schemas, JSON I/O, decision control (clean source)
- [Plugins Reference](sources/clean/code-claude-com-docs-en-plugins-reference-md.md) — official plugins reference: components, manifest schema, CLI, scopes (clean source)

## Summaries

- [Hooks Guide](summaries/hooks-guide.md) — Automate actions with hooks (code.claude.com official guide)
- [Hooks Reference](summaries/hooks.md) — Event schemas, JSON I/O, and decision control (reference doc)
- [Plugins Reference](summaries/plugins-reference.md) — Plugin system: components, scopes, manifest, CLI, versioning (reference doc)

## Concepts

### Hooks

- [Hook Lifecycle Events](concepts/hook-lifecycle-events.md) — all 30+ events and when they fire
- [Hook Types](concepts/hook-types.md) — command, http, mcp_tool, prompt, agent
- [Hook Matchers](concepts/hook-matchers.md) — matcher field, `if` field, per-event filter table
- [Hook Exit Codes](concepts/hook-exit-codes.md) — exit 0/2/other and structured JSON output
- [Hook Scope](concepts/hook-scope.md) — configuration location and permission mode interactions
- [Hook Input and Output](concepts/hook-input-output.md) — JSON input envelope and output fields per event
- [Hook Decision Control](concepts/hook-decision-control.md) — per-event decision patterns reference

### Plugins

- [Plugin Components](concepts/plugin-components.md) — skills, agents, hooks, MCP, LSP, monitors, themes
- [Plugin Manifest Schema](concepts/plugin-manifest-schema.md) — plugin.json field reference
- [Plugin Installation Scopes](concepts/plugin-installation-scopes.md) — user/project/local/managed and skills-dir plugins
- [Plugin Directory Structure](concepts/plugin-directory-structure.md) — standard layout and file locations
- [Plugin Environment Variables](concepts/plugin-environment-variables.md) — CLAUDE_PLUGIN_ROOT, CLAUDE_PLUGIN_DATA, CLAUDE_PROJECT_DIR
- [Plugin CLI Commands](concepts/plugin-cli-commands.md) — init, install, uninstall, enable, disable, update, list, validate
- [Plugin Versioning](concepts/plugin-versioning.md) — explicit vs commit-SHA versioning and cache lifecycle

## Entities

## Comparisons

- [Hooks Guide accuracy check](comparisons/code-claude-com-docs-en-hooks-guide-md.md) — verified wiki pages against source; 2 inaccuracies corrected
- [matcher vs if field](comparisons/matcher-vs-if-field.md) — hook targeting levels: group-level name vs argument-level filtering
- [Sync vs async hooks](comparisons/sync-vs-async-hooks.md) — synchronous vs async: true vs asyncRewake: true execution modes
- [Plugin scopes vs hook scopes](comparisons/plugin-scopes-vs-hook-scopes.md) — same four-tier hierarchy, different capability restrictions
- [Monitors vs command hooks](comparisons/monitors-vs-command-hooks.md) — persistent background stream vs event-triggered single-shot
- [Skills-dir plugins vs marketplace plugins](comparisons/skills-dir-plugins-vs-marketplace-plugins.md) — in-place discovery vs cache copy; path and capability differences

## Answers
