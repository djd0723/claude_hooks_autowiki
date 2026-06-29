---
type: concept
title: "Plugin Directory Structure"
created: 2026-06-29
updated: 2026-06-29
tags: [plugins, directory, structure, layout, file-locations]
source_count: 1
sources:
  - sources/clean/code-claude-com-docs-en-plugins-reference-md.md
---

# Plugin Directory Structure

A complete plugin follows a predictable layout. The manifest is optional; Claude Code auto-discovers components in their default locations.

## Standard layout

```text
enterprise-plugin/
├── .claude-plugin/           # Metadata directory (optional)
│   └── plugin.json           # Plugin manifest
├── skills/                   # Skills with <name>/SKILL.md structure
│   ├── code-reviewer/
│   │   └── SKILL.md
│   └── pdf-processor/
│       ├── SKILL.md
│       └── scripts/
├── commands/                 # Skills as flat .md files
│   ├── status.md
│   └── logs.md
├── agents/                   # Subagent definitions
│   ├── security-reviewer.md
│   ├── performance-tester.md
│   └── compliance-checker.md
├── output-styles/            # Output style definitions
│   └── terse.md
├── themes/                   # Color theme definitions (experimental)
│   └── dracula.json
├── monitors/                 # Background monitor configurations (experimental)
│   └── monitors.json
├── hooks/                    # Hook configurations
│   ├── hooks.json
│   └── security-hooks.json
├── bin/                      # Executables added to PATH
│   └── my-tool
├── settings.json             # Default settings for the plugin
├── .mcp.json                 # MCP server definitions
├── .lsp.json                 # LSP server configurations
└── scripts/                  # Hook and utility scripts
    ├── security-scan.sh
    └── format-code.py
```

## File locations reference

| Component | Default location | Notes |
| :-------- | :--------------- | :---- |
| **Manifest** | `.claude-plugin/plugin.json` | Optional; auto-discovery used when absent |
| **Skills** | `skills/` | `<name>/SKILL.md` structure |
| **Commands** | `commands/` | Flat `.md` files; prefer `skills/` for new plugins |
| **Agents** | `agents/` | Subagent markdown files |
| **Output styles** | `output-styles/` | Output style definitions |
| **Themes** | `themes/` | Color theme JSON files (experimental) |
| **Hooks** | `hooks/hooks.json` | Hook configuration JSON |
| **MCP servers** | `.mcp.json` | MCP server definitions |
| **LSP servers** | `.lsp.json` | Language server configurations |
| **Monitors** | `monitors/monitors.json` | Background monitor configurations (experimental) |
| **Executables** | `bin/` | Invokable as bare commands in Bash tool while plugin is enabled |
| **Settings** | `settings.json` | Default config applied when plugin is enabled (only `agent` and `subagentStatusLine` keys currently supported) |

## Critical rule: components at the plugin root

> **Warning**: The `.claude-plugin/` directory contains only `plugin.json`. All other directories (`commands/`, `agents/`, `skills/`, `output-styles/`, `themes/`, `monitors/`, `hooks/`) must be at the **plugin root**, not inside `.claude-plugin/`.

```text
my-plugin/
├── .claude-plugin/
│   └── plugin.json      ← Only manifest here
├── commands/            ← At root level ✓
├── agents/              ← At root level ✓
└── hooks/               ← At root level ✓
```

**Symptom of mistake**: Plugin loads but components (skills, agents, hooks) are missing.

**Debug checklist**:
1. Run `claude --debug` and look for "loading plugin" messages
2. Check that each component directory is listed in the debug output
3. Verify file permissions allow reading plugin files
4. Run `claude plugin validate ./my-plugin` to check manifest, frontmatter, and hooks.json

## `CLAUDE.md` is not loaded from plugin root

A `CLAUDE.md` file at the plugin root is **not** loaded as project context. Plugins contribute context through skills, agents, and hooks — not CLAUDE.md. To ship instructions that load into Claude's context, put them in a skill.

## `bin/` executables

Files in `bin/` are added to the Bash tool's `PATH` while the plugin is enabled. They are invokable as bare commands in any Bash tool call — no path qualification needed.

## Related concepts

- [Plugin Components](./plugin-components.md) — what each component type does
- [Plugin Manifest Schema](./plugin-manifest-schema.md) — component path fields and manifest configuration
- [Plugin Environment Variables](./plugin-environment-variables.md) — referencing plugin paths in scripts and configs
- [Plugin CLI Commands](./plugin-cli-commands.md) — `validate`, `details`, and other development tools
