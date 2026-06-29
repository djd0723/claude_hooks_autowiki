---
type: concept
title: "Permission Settings"
created: 2026-06-29
updated: 2026-06-29
tags: [settings, permissions, security, rules, modes]
source_count: 1
sources:
  - sources/clean/code-claude-com-docs-en-settings-md.md
---

# Permission Settings

The `permissions` block in [`settings.json`](./settings-files.md) controls which tools Claude Code may use and under what mode it starts. Unlike most settings, permission rules **merge across [scopes](./configuration-scopes.md)** rather than override — managed, user, project, and local rules combine.

> This page covers the settings-side view. The dedicated permissions reference deepens rule syntax, tool-specific patterns, and Bash limitations; see the `permissions.md` source once ingested.

## The permission keys

| Key | Purpose | Example |
| :-- | :------ | :------ |
| `allow` | Rules permitting tool use | `[ "Bash(git diff *)" ]` |
| `ask` | Rules that prompt for confirmation | `[ "Bash(git push *)" ]` |
| `deny` | Rules denying tool use; use to exclude sensitive files | `[ "Read(./.env)", "Bash(curl *)" ]` |
| `additionalDirectories` | Extra working directories for file access | `[ "../docs/" ]` |
| `defaultMode` | Permission mode at startup | `"acceptEdits"` |
| `disableBypassPermissionsMode` | `"disable"` prevents `bypassPermissions` / `--dangerously-skip-permissions` | `"disable"` |
| `skipDangerousModePermissionPrompt` | Skip the confirmation before entering bypass mode (ignored in project settings) | `true` |

`defaultMode` accepts `default`, `acceptEdits`, `plan`, `auto`, `dontAsk`, or `bypassPermissions`. As of v2.1.142, `auto` is **ignored** when set in project or local settings so a repository cannot grant itself auto mode — set it in `~/.claude/settings.json` instead. The `--permission-mode` CLI flag overrides this for one session.

## Permission rule syntax

Rules follow the format `Tool` or `Tool(specifier)`. They are **evaluated in order: deny rules first, then ask, then allow** — the first match determines the outcome regardless of specificity.

| Rule | Effect |
| :--- | :----- |
| `Bash` | Matches all Bash commands |
| `Bash(npm run *)` | Matches commands starting with `npm run` |
| `Read(./.env)` | Matches reading the `.env` file |
| `WebFetch(domain:example.com)` | Matches fetch requests to example.com |

Tool-name globs are supported only in the tool position after a literal `mcp__<server>__` prefix (e.g. `mcp__github__get_*`); the server segment must be glob-free. In `deny`, tool names accept globs — `"*"` denies every tool and `"mcp__*"` denies all MCP tools.

## Exclude sensitive files

`permissions.deny` is the supported way to keep Claude away from secrets (it replaces the deprecated `ignorePatterns`). Matching files are excluded from file discovery and search, and read operations on them are denied:

```json
{
  "permissions": {
    "deny": [
      "Read(./.env)",
      "Read(./.env.*)",
      "Read(./secrets/**)",
      "Read(./config/credentials.json)"
    ]
  }
}
```

## Related concepts

- [Settings Files](./settings-files.md) — where the `permissions` block lives
- [Settings Precedence](./settings-precedence.md) — why permission arrays merge instead of override
- [Configuration Scopes](./configuration-scopes.md) — managed scope can lock permission rules org-wide
- [Sandbox Settings](./sandbox-settings.md) — `Edit`/`Read`/`WebFetch` rules also feed sandbox filesystem and network restrictions
- [Tool Permission Rules](./tool-permission-rules.md) — the `ToolName(specifier)` format and per-tool specifier table
- [Built-in Tools](./built-in-tools.md) — the catalog of tool names `allow`/`ask`/`deny` reference
