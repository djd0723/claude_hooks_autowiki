---
type: summary
title: "Settings — files, precedence, and scopes"
slug: code-claude-com-docs-en-settings-md
created: 2026-06-29
updated: 2026-06-29
tags: [settings, configuration, precedence, scopes, managed-settings, sandbox, environment-variables]
sources:
  - sources/clean/code-claude-com-docs-en-settings-md.md
---

# Summary: Claude Code settings

The authoritative reference for how Claude Code is configured: the scope system, the settings files and their locations, the precedence hierarchy, the full catalog of `settings.json` keys, environment variables, sandbox and permission settings, and working directories.

## Key thesis from source

> "Claude Code offers a variety of settings to configure its behavior to meet your needs. You can configure Claude Code by running the `/config` command, which opens a tabbed Settings interface where you can view status information and modify configuration options."

> "The `settings.json` file is the official mechanism for configuring Claude Code through hierarchical settings"

## Configuration scopes

Claude Code uses a scope system to decide where a configuration applies and who it is shared with. See [configuration scopes](../concepts/configuration-scopes.md).

| Scope | Location | Who it affects | Shared with team? |
| :---- | :------- | :------------- | :---------------- |
| Managed | Server-managed settings, plist / registry, or system-level `managed-settings.json` | All org members (server) / all machine users (plist, HKLM, file) / current user (HKCU) | Yes (deployed by IT) |
| User | `~/.claude/` directory | You, across all projects | No |
| Project | `.claude/` in repository | All collaborators on this repository | Yes (committed to git) |
| Local | `.claude/settings.local.json` | You, in this repository only | No (gitignored when Claude Code creates it) |

On Windows, `~/.claude` resolves to `%USERPROFILE%\.claude`.

What uses scopes (per-feature locations):

| Feature | User | Project | Local |
| :------ | :--- | :------ | :---- |
| Settings | `~/.claude/settings.json` | `.claude/settings.json` | `.claude/settings.local.json` |
| Subagents | `~/.claude/agents/` | `.claude/agents/` | None |
| MCP servers | `~/.claude.json` | `.mcp.json` | `~/.claude.json` (per-project) |
| Plugins | `~/.claude/settings.json` | `.claude/settings.json` | `.claude/settings.local.json` |
| CLAUDE.md | `~/.claude/CLAUDE.md` | `CLAUDE.md` or `.claude/CLAUDE.md` | `CLAUDE.local.md` |

## Settings files

See [settings files](../concepts/settings-files.md) and [managed settings](../concepts/managed-settings.md).

- **User settings**: `~/.claude/settings.json`, all projects.
- **Project settings**: `.claude/settings.json` (checked in) and `.claude/settings.local.json` (not checked in; gitignored when Claude Code creates it).
- **Managed settings**: centralized control via multiple delivery mechanisms, all using the same JSON format and not overridable by user or project:
  - **Server-managed** — from Anthropic's servers via the Claude.ai admin console.
  - **MDM/OS-level policies** — macOS `com.anthropic.claudecode` managed preferences domain; Windows `HKLM\SOFTWARE\Policies\ClaudeCode` registry `Settings` value; Windows user-level `HKCU\SOFTWARE\Policies\ClaudeCode` (lowest policy priority).
  - **File-based** `managed-settings.json` / `managed-mcp.json` in system dirs:

| Platform | File-based managed settings directory |
| :------- | :------------------------------------ |
| macOS | `/Library/Application Support/ClaudeCode/` |
| Linux and WSL | `/etc/claude-code/` |
| Windows | `C:\Program Files\ClaudeCode\` |

  The legacy Windows path `C:\ProgramData\ClaudeCode\managed-settings.json` is unsupported as of v2.1.75. A drop-in directory `managed-settings.d/` allows independent policy fragments: `managed-settings.json` merges first as the base, then `*.json` drop-ins are sorted alphabetically and merged on top (scalars overridden, arrays concatenated + deduped, objects deep-merged). Use numeric prefixes (`10-telemetry.json`, `20-security.json`) to control order.
- **Other configuration**: `~/.claude.json` holds the OAuth session, user/local-scope MCP configs, per-project state, and caches. Project MCP servers live in `.mcp.json`.

Claude Code retains the five most recent timestamped backups of config files. Adding the `$schema` line (`https://json.schemastore.org/claude-code-settings.json`) enables editor autocomplete/validation.

### When edits take effect

Settings files are watched and reloaded live for most keys (`permissions`, `hooks`, `apiKeyHelper`), and the `ConfigChange` hook fires per detected change. Read-once keys applied on next restart: `model` (use `/model` mid-session) and `outputStyle`.

### Invalid entries in managed settings

> "Managed settings parse tolerantly. When a managed configuration contains an entry that fails schema validation, Claude Code strips that entry, records a warning, and enforces every remaining valid policy. A single typo cannot disable the rest of your organization's policy."

Security-enforcement fields fail safe per field: `allowedMcpServers`/`availableModels` become empty allowlists; `allowManagedMcpServersOnly`/`enforceAvailableModels` treated as `true`; `forceLoginOrgUUID` blocks all logins until fixed. `requiredMinimumVersion`/`requiredMaximumVersion` fail open (stripped, not enforced) so a bad push can't block startup. Tolerance applies only to managed settings — user/project/local files are rejected as a whole on validation failure. Errors surface in the startup dialog, `-p` stderr, and `claude doctor`.

## Settings precedence

See [settings precedence](../concepts/settings-precedence.md). From highest to lowest:

| Rank | Layer | Notes |
| :--- | :---- | :---- |
| 1 (highest) | Managed settings | Cannot be overridden, including by CLI args |
| 2 | Command line arguments | `--settings <file-or-json>`; temporary session overrides |
| 3 | Local project settings | `.claude/settings.local.json` |
| 4 | Shared project settings | `.claude/settings.json` (in source control) |
| 5 (lowest) | User settings | `~/.claude/settings.json` |

Within the managed tier: server-managed > MDM/OS-level policies > file-based (`managed-settings.d/*.json` + `managed-settings.json`) > HKCU registry (Windows only). Only one managed source is used; tiers do not merge.

> "**Array settings merge across scopes.** When the same array-valued setting (such as `sandbox.filesystem.allowWrite` or `permissions.allow`) appears in multiple scopes, the arrays are **concatenated and deduplicated**, not replaced."

Two array exceptions: `fallbackModel` (highest-precedence file supplies the whole ordered chain) and `availableModels` (a managed-defined list applies as-is). `parentSettingsBehavior` (`"first-wins"` default / `"merge"`) controls whether SDK/IDE-supplied managed settings apply under an admin tier.

Run `/status` (Status tab, `Setting sources` line) to see which layers loaded; `/doctor` shows file errors.

## Notable settings keys

`settings.json` supports many options. A representative subset, mirroring how the source lists `Key | Description | Example`:

| Key | Description | Example |
| :-- | :---------- | :------ |
| `apiKeyHelper` | Custom command that generates an auth value sent as `X-Api-Key` / `Authorization: Bearer` | `/bin/generate_temp_api_key.sh` |
| `attribution` | Customize git commit / PR attribution | `{"commit": "...", "pr": ""}` |
| `autoCompactEnabled` | Auto-compact conversation near the context limit (default `true`) | `false` |
| `cleanupPeriodDays` | Days before old session files are deleted (default `30`, min `1`) | `20` |
| `disableAllHooks` | Disable all hooks and custom status line | `true` |
| `env` | Environment variables applied to every session and spawned subprocesses | `{"FOO": "bar"}` |
| `fallbackModel` | Ordered fallback model chain (cap 3); does not merge across files | `["claude-sonnet-4-6", "claude-haiku-4-5"]` |
| `hooks` | Configure lifecycle-event commands | See [hooks](hooks.md) |
| `model` | Override the default model (read at startup) | `"claude-sonnet-4-6"` |
| `outputStyle` | Output style adjusting the system prompt | `"Explanatory"` |
| `permissions` | Allow/ask/deny rules, directories, default mode (see below) | — |
| `statusLine` | Custom status line command | `{"type": "command", "command": "~/.claude/statusline.sh"}` |
| `theme` | Interface color theme (default `"dark"`) | `"dark"` |
| `enabledPlugins` | `"plugin@marketplace": true/false` enablement map | — |
| `extraKnownMarketplaces` | Additional plugin marketplaces for the repo | — |

Managed-only governance keys include `allowManagedHooksOnly`, `allowManagedMcpServersOnly`, `allowManagedPermissionRulesOnly`, `allowedMcpServers`/`deniedMcpServers`, `strictKnownMarketplaces`, `blockedMarketplaces`, `strictPluginOnlyCustomization`, `forceLoginMethod`/`forceLoginOrgUUID`, `requiredMinimumVersion`/`requiredMaximumVersion`, `policyHelper`, and `parentSettingsBehavior`.

### Global config settings (`~/.claude.json`, not `settings.json`)

`autoConnectIde`, `autoInstallIdeExtension`, `externalEditorContext`, `teammateDefaultModel`. Putting these in `settings.json` triggers a schema validation error.

## Permission settings

See [permission settings](../concepts/permission-settings.md). Under `permissions`:

| Key | Description | Example |
| :-- | :---------- | :------ |
| `allow` | Rules allowing tool use | `[ "Bash(git diff *)" ]` |
| `ask` | Rules asking for confirmation | `[ "Bash(git push *)" ]` |
| `deny` | Rules denying tool use / excluding sensitive files | `[ "Read(./.env)", "Bash(curl *)" ]` |
| `additionalDirectories` | Extra working directories for file access | `[ "../docs/" ]` |
| `defaultMode` | Default permission mode (`auto` ignored in project/local as of v2.1.142) | `"acceptEdits"` |
| `disableBypassPermissionsMode` | `"disable"` blocks `--dangerously-skip-permissions` | `"disable"` |
| `skipDangerousModePermissionPrompt` | Skip bypass-mode confirmation (ignored in project settings) | `true` |

Rules follow `Tool` or `Tool(specifier)`; evaluation order is deny, then ask, then allow, first match wins. `permissions.deny` replaces the deprecated `ignorePatterns` for excluding sensitive files.

## Sandbox settings

See [sandbox settings](../concepts/sandbox-settings.md). Under `sandbox`, sandboxing isolates bash commands from filesystem and network (macOS, Linux, WSL2).

| Key | Description |
| :-- | :---------- |
| `enabled` | Enable bash sandboxing (default `false`) |
| `failIfUnavailable` | Exit at startup if the sandbox can't start (hard gate) |
| `autoAllowBashIfSandboxed` | Auto-approve bash when sandboxed (default `true`) |
| `excludedCommands` | Commands that run outside the sandbox |
| `allowUnsandboxedCommands` | Allow `dangerouslyDisableSandbox` escape hatch (default `true`) |
| `filesystem.allowWrite` / `denyWrite` | Sandbox write paths (merged across scopes; merged with `Edit(...)` rules) |
| `filesystem.denyRead` / `allowRead` | Sandbox read paths (`allowRead` overrides `denyRead`; merged with `Read(...)` rules) |
| `credentials.files` / `credentials.envVars` | Credential files / env vars blocked in sandbox (deny only; v2.1.187+) |
| `network.allowedDomains` / `deniedDomains` | Outbound domain allow/deny (wildcards; deny precedence) |
| `network.allowUnixSockets` / `allowAllUnixSockets` / `allowLocalBinding` / `allowMachLookup` | Socket, localhost-binding, and Mach-lookup access |
| `network.httpProxyPort` / `socksProxyPort` | Bring-your-own proxy ports |
| `enableWeakerNestedSandbox` / `enableWeakerNetworkIsolation` / `allowAppleEvents` | Security-reducing escape hatches |
| `bwrapPath` / `socatPath` | (Managed only, Linux/WSL2) absolute binary paths |

**Sandbox path prefixes** (`filesystem.*` and `credentials.files`): `/` absolute; `~/` home-relative; `./` or no prefix is project-root-relative for project settings, or `~/.claude`-relative for user settings. The older `//path` absolute prefix still works. This differs from Read/Edit permission rules, which use `//path` for absolute and `/path` for project-relative.

## Environment variables

Any environment variable can be set under the `env` key in `settings.json` to apply to every session and to spawned subprocesses; see the full list in the env-vars reference. As of v2.1.143, `NO_COLOR`/`FORCE_COLOR` set under `env` pass to subprocesses but do not change Claude Code's own interface colors (set those in your shell). Many settings have equivalent env vars (e.g., `DISABLE_AUTO_COMPACT`, `CLAUDE_CODE_DISABLE_AUTO_MEMORY`, `DISABLE_AUTOUPDATER`).

## Worktree settings

Configure how `--worktree` creates git worktrees: `worktree.baseRef` (`"fresh"` default / `"head"`), `worktree.symlinkDirectories`, `worktree.sparsePaths`, and `worktree.bgIsolation` (`"worktree"` default / `"none"`, v2.1.143+).

## Working directories, subagents, and other configuration surfaces

See [working directories](../concepts/working-directories.md). `permissions.additionalDirectories` grants extra file-access directories, though most `.claude/` configuration is not discovered from them. Subagents live in `~/.claude/agents/` and `.claude/agents/`. The `statusLine` setting and `footerLinksRegexes` badges render in the UI; see [statusline / context / backup](../concepts/statusline-context-backup.md). Default plugin enablement (when no `enabledPlugins` entry exists at any scope) falls back to a plugin's `defaultEnabled` value; see [plugin default settings](../concepts/plugin-default-settings.md).

## Relationship to concept pages

This source is the primary backing for:
- [settings files](../concepts/settings-files.md) — file types, locations, managed delivery mechanisms, backups
- [settings precedence](../concepts/settings-precedence.md) — the five-layer hierarchy and array-merge rules
- [configuration scopes](../concepts/configuration-scopes.md) — managed/user/project/local scope model and per-feature locations
- [managed settings](../concepts/managed-settings.md) — managed-only keys, tolerant parsing, policy helpers
- [sandbox settings](../concepts/sandbox-settings.md) — `sandbox.*` filesystem/network/credentials keys and path prefixes
- [working directories](../concepts/working-directories.md) — `additionalDirectories` and file-access scope
- [plugin default settings](../concepts/plugin-default-settings.md) — `enabledPlugins` fallback and marketplace config
- [statusline / context / backup](../concepts/statusline-context-backup.md) — status line, footer badges, config backups
- [permission settings](../concepts/permission-settings.md) — `permissions.*` keys and rule syntax

Related summary: [hooks reference](hooks.md) (the `hooks` key and managed-hook governance).
