---
type: concept
title: "Settings Files"
created: 2026-06-29
updated: 2026-06-29
tags: [settings, configuration, managed, mdm, json]
source_count: 1
sources:
  - sources/clean/code-claude-com-docs-en-settings-md.md
---

# Settings Files

> The `settings.json` file is the official mechanism for configuring Claude Code through hierarchical settings.

Each [scope](./configuration-scopes.md) is backed by a concrete file (or, for managed settings, a delivery mechanism). This page covers where those files live, how they reload, and how managed settings tolerate errors.

## File locations

- **User settings** — `~/.claude/settings.json`, applies to all projects.
- **Project settings** — saved in the project directory:
  - `.claude/settings.json` — checked into source control, shared with the team.
  - `.claude/settings.local.json` — not checked in, for personal preferences and experimentation. When Claude Code creates this file it configures git to ignore it; if you create it yourself, add it to `.gitignore` manually.
- **Managed settings** — centralized, cannot be overridden by user or project settings (see below).
- **Other configuration** — `~/.claude.json` holds the OAuth session, user/local-scope MCP server configs, per-project state (allowed tools, trust settings), and caches. Project-scoped MCP servers live separately in `.mcp.json`.

Claude Code keeps timestamped backups of configuration files, retaining the five most recent to prevent data loss.

### The `$schema` line

Pointing `settings.json` at the [official JSON schema](https://json.schemastore.org/claude-code-settings.json) enables autocomplete and inline validation in VS Code, Cursor, and other schema-aware editors:

```json
{
  "$schema": "https://json.schemastore.org/claude-code-settings.json"
}
```

The published schema updates periodically and may lag the latest CLI releases, so a validation warning on a recently documented field does not necessarily mean the config is invalid.

## Managed settings delivery mechanisms

All managed mechanisms use the same JSON format and cannot be overridden by user or project settings:

- **Server-managed** — delivered from Anthropic's servers via the Claude.ai admin console.
- **MDM / OS-level policies** — native device management:
  - macOS: `com.anthropic.claudecode` managed preferences domain (plist keys mirror `managed-settings.json`).
  - Windows: `HKLM\SOFTWARE\Policies\ClaudeCode` registry key with a `Settings` value containing JSON.
  - Windows (user-level): `HKCU\SOFTWARE\Policies\ClaudeCode` — lowest policy priority, used only when no admin source exists.
- **File-based** — `managed-settings.json` (and `managed-mcp.json`) deployed to system directories:
  - macOS: `/Library/Application Support/ClaudeCode/`
  - Linux and WSL: `/etc/claude-code/`
  - Windows: `C:\Program Files\ClaudeCode\` (the legacy `C:\ProgramData\ClaudeCode\` path is unsupported as of v2.1.75)

File-based managed settings also support a `managed-settings.d/` drop-in directory so independent teams can deploy policy fragments without editing one shared file. Following the systemd convention, `managed-settings.json` merges first as the base, then `*.json` files in the drop-in directory merge alphabetically on top — scalars override, arrays concatenate and de-duplicate, objects deep-merge. Use numeric prefixes (`10-telemetry.json`, `20-security.json`) to control order.

## When edits take effect

Claude Code watches settings files and reloads them on change, so most keys — including `permissions`, `hooks`, and credential helpers like `apiKeyHelper` — apply to the running session without a restart. Reload covers all scopes, and the [`ConfigChange` hook](./hook-lifecycle-events.md) fires for each detected change.

A few keys are read once at session start and apply on the next restart:

- `model` — use `/model` to switch mid-session.
- `outputStyle` — part of the system prompt, rebuilt on `/clear` or restart.

## Invalid entries in managed settings

Managed settings parse **tolerantly** (v2.1.169+): an entry that fails schema validation is stripped, a warning is recorded, and every remaining valid policy is still enforced. A single typo cannot disable the rest of an organization's policy. This is consistent across server-managed, plist/registry, and file delivery.

Security-enforcement fields are handled per field — when present but invalid they **fail closed** rather than being silently dropped. For example:

| Field | Behavior when present but invalid |
| :---- | :-------------------------------- |
| `allowedMcpServers` | Enforced as an empty allowlist (no MCP servers admitted) until fixed |
| `allowManagedMcpServersOnly` | Treated as `true` |
| `forceLoginOrgUUID` | No organization permitted to log in until fixed |

By contrast, `requiredMinimumVersion` / `requiredMaximumVersion` **fail open** by design — an invalid value is stripped so a bad policy push cannot prevent Claude Code from starting.

Validation errors surface in three places: a startup dialog (interactive), stderr (headless `-p` runs), and `claude doctor` (lists each entry with its source and field). This tolerance applies **only** to managed settings — user, project, and local files remain strict and are rejected as a whole if they fail validation.

## Related concepts

- [Configuration Scopes](./configuration-scopes.md) — which audience each file serves
- [Settings Precedence](./settings-precedence.md) — how the layers combine when a key appears in several
- [Permission Settings](./permission-settings.md) — the most common keys in a `settings.json`
- [Sandbox Settings](./sandbox-settings.md) — the `sandbox` configuration block
- [SDK Setting Sources](./sdk-setting-sources.md) — how the Agent SDK chooses which of these files to load
