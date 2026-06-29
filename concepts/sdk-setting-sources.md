---
type: concept
title: "SDK Setting Sources"
created: 2026-06-29
updated: 2026-06-29
tags: [agent-sdk, settingSources, filesystem, configuration, multi-tenant, isolation]
source_count: 1
sources:
  - sources/clean/code-claude-com-docs-en-agent-sdk-claude-code-features-md.md
---

# SDK Setting Sources

> The Agent SDK is built on the same foundation as Claude Code, which means your SDK agents have access to the same filesystem-based features: project instructions (`CLAUDE.md` and rules), skills, hooks, and more.

The `settingSources` option (`setting_sources` in Python, `settingSources` in TypeScript) is the single switch that decides **which filesystem-based settings the SDK loads** into an agent. By default the SDK behaves like the CLI; passing this option lets you opt in to specific sources or shut filesystem loading off entirely.

## The default vs. the empty array

> When you omit `settingSources`, `query()` reads the same filesystem settings as the Claude Code CLI: user, project, and local settings, CLAUDE.md files, and `.claude/` skills, agents, and commands. To run without these, pass `settingSources: []`, which limits the agent to what you configure programmatically.

Omitting `settingSources` is equivalent to `["user", "project", "local"]`. Pass an explicit list to load only some sources, or `[]` to disable user, project, and local settings and run on purely programmatic configuration.

## The three sources

Each source loads from a specific location, where `<cwd>` is the working directory you pass via the `cwd` option (or the process's current directory if unset).

| Source | What it loads | Location |
| :----- | :------------ | :------- |
| `"project"` | Project CLAUDE.md, `.claude/rules/*.md`, project skills, project hooks, project `settings.json` | `<cwd>/.claude/` for `settings.json` and hooks; `<cwd>` and every parent directory for CLAUDE.md and rules; `<cwd>` and every parent up to the repository root for skills |
| `"user"` | User CLAUDE.md, `~/.claude/rules/*.md`, user skills, user settings | `~/.claude/` |
| `"local"` | CLAUDE.local.md, `.claude/settings.local.json` | `<cwd>/.claude/` for `settings.local.json`; `<cwd>` and every parent directory for CLAUDE.local.md |

The `cwd` option determines where the SDK looks for project-level inputs. CLAUDE.md and rules load from `<cwd>` and every parent directory; skills load from `<cwd>` and every parent up to the repository root; but **project `settings.json` and hooks load only from `<cwd>/.claude/` with no parent-directory fallback.**

## What `settingSources` does not control

`settingSources` covers only user, project, and local settings. A few inputs are read **regardless of its value** — the security-critical detail for anyone deploying the SDK:

| Input | Behavior | To disable |
| :---- | :------- | :--------- |
| Managed policy settings | Endpoint-managed policy (MDM plist, registry policy, managed settings file) loads from the host. Server-managed settings are fetched on an eligible configuration when the session authenticates with an org OAuth login or a directly configured API key | Endpoint policy: remove the managed file/plist/registry policy. Server-managed: controlled by your org admin; cannot be disabled from the SDK |
| `~/.claude.json` global config | Always read | Relocate with `CLAUDE_CONFIG_DIR` in `env` |
| Auto memory at `~/.claude/projects/<project>/memory/` | Loaded by default into the system prompt | `autoMemoryEnabled: false` in settings, or `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1` in `env` |
| claude.ai MCP connectors | Loaded when the active auth method is a claude.ai subscription. Passing `mcpServers: {}` does not suppress them | `strictMcpConfig: true`, `disableClaudeAiConnectors: true` in settings, or `ENABLE_CLAUDEAI_MCP_SERVERS=false` in `env` |

## Multi-tenant isolation warning

> Do not rely on default `query()` options for multi-tenant isolation. Because the inputs above are read regardless of `settingSources`, an SDK process can pick up host-level configuration and per-directory memory. For multi-tenant deployments, run each tenant in its own filesystem and set `settingSources: []` plus `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1` in `env`. Server-managed settings are fetched when the process authenticates with an organization credential; filesystem isolation does not remove them.

The practical takeaway: `settingSources: []` is necessary but **not sufficient** for tenant isolation — combine it with a per-tenant filesystem, the auto-memory disable, and awareness that server-managed settings ride along with the org credential.

## Related concepts

- [SDK Project Instructions](./sdk-project-instructions.md) — how CLAUDE.md and rules load when `"project"` is included
- [SDK Skills Loading](./sdk-skills-loading.md) — skill discovery gated by `settingSources` plus the `skills` option
- [SDK Callback Hooks](./sdk-callback-hooks.md) — filesystem hooks load via `settingSources`, running side by side with programmatic callbacks
- [SDK Extension Features](../comparisons/sdk-extension-features.md) — pick the right extension surface for a goal
- [Settings Files](./settings-files.md) — the `settings.json` files and managed-delivery mechanisms the SDK reads
- [Configuration Scopes](./configuration-scopes.md) — the user/project/local scopes mirrored by these sources
