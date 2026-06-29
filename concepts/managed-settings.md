---
type: concept
title: "Managed Settings"
created: 2026-06-29
updated: 2026-06-29
tags: [permissions, managed, enterprise, policy, lockdown, security]
source_count: 1
sources:
  - sources/clean/code-claude-com-docs-en-permissions-md.md
---

# Managed Settings

For organizations needing centralized control, administrators deploy **managed settings** that can't be overridden by user or project settings. They follow the same format as regular settings and are delivered via MDM/OS-level policies, managed settings files, or server-managed settings — see [Configuration Scopes](./configuration-scopes.md) for the managed scope's locations and [Settings Precedence](./settings-precedence.md) for why managed wins over everything (including CLI arguments).

This page covers the **managed-only** lock-down keys — settings that have no effect unless placed in managed settings.

## Managed-only settings

| Setting | Effect |
| :------ | :----- |
| `allowAllClaudeAiMcps` | When `true`, claude.ai connectors load alongside a deployed `managed-mcp.json` instead of being suppressed |
| `allowedChannelPlugins` | Allowlist of channel plugins that may push messages (replaces the default Anthropic allowlist; requires `channelsEnabled: true`) |
| `allowManagedHooksOnly` | When `true`, only managed hooks, SDK hooks, and hooks from force-enabled managed plugins load; all other [hooks](./hook-lifecycle-events.md) are blocked |
| `allowManagedMcpServersOnly` | When `true`, only `allowedMcpServers` from managed settings are respected (`deniedMcpServers` still merges from all sources) |
| `allowManagedPermissionRulesOnly` | When `true`, user/project settings can't define `allow`/`ask`/`deny` [permission rules](./permission-evaluation.md) — only managed rules apply |
| `blockedMarketplaces` | Blocklist of marketplace sources, checked before download so they never touch the filesystem |
| `channelsEnabled` | Allow channels for the organization |
| `forceRemoteSettingsRefresh` | When `true`, blocks CLI startup until remote managed settings are freshly fetched; exits if the fetch fails |
| `pluginTrustMessage` | Custom message appended to the plugin trust warning shown before installation |
| `sandbox.filesystem.allowManagedReadPathsOnly` | When `true`, only managed `filesystem.allowRead` paths are respected (`denyRead` still merges) |
| `sandbox.network.allowManagedDomainsOnly` | When `true`, only managed `allowedDomains` and `WebFetch(domain:...)` allow rules apply; non-allowed domains are blocked without prompting |
| `strictKnownMarketplaces` | Controls which plugin marketplace sources users can add and install from |
| `strictPluginOnlyCustomization` | Blocks skills, agents, hooks, and MCP servers from user/project sources so they come only from plugins or managed settings (`true` locks all four; an array locks named ones) |
| `wslInheritsWindowsSettings` | When `true` in the Windows policy chain, WSL also reads managed settings from the Windows policy |

## The two-sided merge pattern

Several keys above follow the same shape: the **allow** side becomes managed-only while the **deny** side keeps merging from every scope. `allowManagedMcpServersOnly`, `sandbox.filesystem.allowManagedReadPathsOnly`, and `sandbox.network.allowManagedDomainsOnly` all let admins fix the allowlist centrally while still letting any scope add restrictions — consistent with the deny-first precedence in [Permission Evaluation](./permission-evaluation.md).

## Mode lock-downs

`disableBypassPermissionsMode` (and `disableAutoMode`) are "typically placed in managed settings to enforce organizational policy, but [they] work from any scope." See [Permission Modes](./permission-modes.md) for what each disables. On Team and Enterprise plans, an Owner enables or disables Remote Control and web sessions org-wide in Claude Code admin settings; Remote Control can also be disabled per device via `disableRemoteControl`.

## Related concepts

- [Configuration Scopes](./configuration-scopes.md) — the managed scope and its delivery mechanisms
- [Settings Precedence](./settings-precedence.md) — why managed settings can't be overridden
- [Permission Evaluation](./permission-evaluation.md) — `allowManagedPermissionRulesOnly` and the deny-first merge
- [Permission Modes](./permission-modes.md) — the bypass/auto mode disable flags
- [Sandbox Settings](./sandbox-settings.md) — the managed-only filesystem/network allowlist flags
