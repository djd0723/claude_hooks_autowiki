---
type: summary
title: "Claude Code features in the SDK"
slug: code-claude-com-docs-en-agent-sdk-claude-code-features-md
created: 2026-06-29
updated: 2026-06-29
tags: [agent-sdk, features, claude-md, skills, hooks, setting-sources, configuration]
sources:
  - sources/clean/code-claude-com-docs-en-agent-sdk-claude-code-features-md.md
---

# Summary: Claude Code features in the SDK

How the Agent SDK exposes the same filesystem-based features as the Claude Code CLI — project instructions (CLAUDE.md and rules), skills, hooks, subagents, and MCP — and how each is turned on or off through `query()` options.

## Key thesis from source

> "The Agent SDK is built on the same foundation as Claude Code, which means your SDK agents have access to the same filesystem-based features: project instructions (`CLAUDE.md` and rules), skills, hooks, and more."

The master switch is `settingSources`:

> "When you omit `settingSources`, `query()` reads the same filesystem settings as the Claude Code CLI: user, project, and local settings, CLAUDE.md files, and `.claude/` skills, agents, and commands. To run without these, pass `settingSources: []`, which limits the agent to what you configure programmatically. Managed policy settings and the global `~/.claude.json` config are read regardless of this option."

## What this source covers

- `settingSources` (Python `setting_sources`) — opt in to filesystem settings, or pass `[]` to disable
- Per-source load tables and the role of `cwd`
- Inputs read regardless of `settingSources` (managed policy, global config, auto memory, claude.ai connectors) and the multi-tenant isolation warning
- [Project instructions](../concepts/sdk-project-instructions.md): CLAUDE.md and `.claude/rules/*.md` load locations
- [Skills loading](../concepts/sdk-skills-loading.md): discovery via `settingSources` plus the `skills` option
- Filesystem vs programmatic [callback hooks](../concepts/sdk-callback-hooks.md)
- A "Choose the right feature" goal-to-surface mapping table

## Setting sources

The `settingSources` option (TS) / `setting_sources` (Python) controls which filesystem-based settings load. Pass an explicit list to opt in, or `[]` to disable user, project, and local settings. Each source maps to a location relative to `<cwd>`:

| Source | What it loads | Location |
| :----- | :------------ | :------- |
| `"project"` | Project CLAUDE.md, `.claude/rules/*.md`, project skills, project hooks, project `settings.json` | `<cwd>/.claude/` for `settings.json` and hooks; `<cwd>` and every parent dir for CLAUDE.md and rules; `<cwd>` and every parent dir up to repo root for skills |
| `"user"` | User CLAUDE.md, `~/.claude/rules/*.md`, user skills, user settings | `~/.claude/` |
| `"local"` | CLAUDE.local.md, `.claude/settings.local.json` | `<cwd>/.claude/` for `settings.local.json`; `<cwd>` and every parent dir for CLAUDE.local.md |

> "Omitting `settingSources` is equivalent to `["user", "project", "local"]`."

The `cwd` option determines where project-level inputs are sought. CLAUDE.md and rules load from `<cwd>` and every parent directory; skills load from `<cwd>` up to the repository root; project `settings.json` and hooks load only from `<cwd>/.claude/` with no parent fallback. See [setting sources](../concepts/sdk-setting-sources.md).

### What settingSources does NOT control

A few inputs are read regardless of `settingSources`:

| Input | Behavior | To disable |
| :---- | :------- | :--------- |
| Managed policy settings | Endpoint-managed (MDM plist, registry, managed file) loads from host; server-managed settings fetched on org OAuth / API key auth | Remove the host policy file; server-managed is org-admin controlled and cannot be disabled from the SDK |
| `~/.claude.json` global config | Always read | Relocate with `CLAUDE_CONFIG_DIR` in `env` |
| Auto memory (`~/.claude/projects/<project>/memory/`) | Loaded by default into the system prompt | `autoMemoryEnabled: false`, or `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1` in `env` |
| claude.ai MCP connectors | Loaded under a claude.ai subscription auth; `mcpServers: {}` does not suppress them | `strictMcpConfig: true`, `disableClaudeAiConnectors: true`, or `ENABLE_CLAUDEAI_MCP_SERVERS=false` |

> "Do not rely on default `query()` options for multi-tenant isolation. Because the inputs above are read regardless of `settingSources`, an SDK process can pick up host-level configuration and per-directory memory. For multi-tenant deployments, run each tenant in its own filesystem and set `settingSources: []` plus `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1` in `env`."

See [statusline/context backup](../concepts/statusline-context-backup.md) for how context costs and host-level inputs accumulate.

## Project instructions (CLAUDE.md and rules)

When `settingSources` includes `"project"`, the SDK loads `CLAUDE.md` and `.claude/rules/*.md` into context at session start, so the agent follows project conventions without repeating them in every prompt. Load is gated by source:

| Level | Location | When loaded |
| :---- | :------- | :---------- |
| Project (root) | `<cwd>/CLAUDE.md` or `<cwd>/.claude/CLAUDE.md` | `"project"` |
| Project rules | `<cwd>/.claude/rules/*.md` and in every parent dir | `"project"` |
| Project (parent dirs) | `CLAUDE.md` above `cwd` | `"project"`, at session start |
| Project (child dirs) | `CLAUDE.md` in subdirectories of `cwd` | `"project"`, on demand when the agent reads a file in that subtree |
| Local | `<cwd>/CLAUDE.local.md` and in every parent dir | `"local"` |
| User | `~/.claude/CLAUDE.md` | `"user"` |
| User rules | `~/.claude/rules/*.md` | `"user"` |

All levels are additive; there is no hard precedence between levels, so conflicts resolve however Claude interprets them. State precedence explicitly in the more specific file. Context can also be injected via `systemPrompt` without CLAUDE.md — use CLAUDE.md when the same context should be shared between interactive sessions and SDK agents. See [project instructions](../concepts/sdk-project-instructions.md).

## Skills

Skills are markdown files that load on demand: the agent sees descriptions at startup and loads full content when relevant.

> "Skills are discovered from the filesystem through `settingSources`. When the `skills` option on `query()` is omitted, discovered user and project skills are enabled and the Skill tool is available, matching CLI behavior. To control which skills are enabled, pass `skills` as `"all"`, a list of skill names, or `[]` to disable all."

When `skills` is set, the SDK adds the Skill tool to `allowedTools` automatically; if you pass an explicit `tools` list, include `"Skill"` so Claude can invoke skills.

> "Skills must be created as filesystem artifacts (`.claude/skills/<name>/SKILL.md`). The SDK does not have a programmatic API for registering skills."

See [skills loading](../concepts/sdk-skills-loading.md).

## Hooks

The SDK supports two hook mechanisms that run side by side during the same lifecycle:

- **Filesystem hooks** — shell commands in `settings.json`, loaded when `settingSources` includes the relevant source; identical to interactive-session hooks.
- **Programmatic hooks** — callback functions passed directly to `query()` via the `hooks` parameter, running in your application process and able to return structured decisions.

> "If you already have hooks in your project's `.claude/settings.json` and you set `settingSources: ["project"]`, those hooks run automatically in the SDK with no extra configuration."

Hook callbacks receive the tool input and return a decision dict. Returning `{}` allows the tool; to block, return a `hookSpecificOutput` with `permissionDecision: "deny"` and a `permissionDecisionReason` (sent to Claude as the tool result). The top-level `decision`/`reason` fields are deprecated for `PreToolUse`.

### When to use which hook type

| Hook type | Best for |
| :-------- | :------- |
| **Filesystem** (`settings.json`) | Sharing hooks between CLI and SDK. Supports `"command"`, `"http"`, `"mcp_tool"`, `"prompt"`, and `"agent"` handlers. Fire in the main agent and any subagents |
| **Programmatic** (callbacks in `query()`) | Application-specific logic, structured decisions, in-process integration. Also fire inside subagents; the callback receives `agent_id` and `agent_type` |

> "The TypeScript SDK supports additional hook events beyond Python, including `SessionStart`, `SessionEnd`, `TeammateIdle`, and `TaskCompleted`."

See [callback hooks](../concepts/sdk-callback-hooks.md) and the [hooks reference summary](hooks.md).

## Choose the right feature

The doc closes with a goal-to-surface map:

| You want to... | Use | SDK surface |
| :------------- | :-- | :---------- |
| Set project conventions the agent always follows | CLAUDE.md | `settingSources: ["project"]` loads it automatically |
| Give reference material loaded when relevant | Skills | `settingSources` + `skills` option |
| Run a reusable workflow (deploy, review, release) | User-invocable skills | `settingSources` + `skills` option |
| Delegate an isolated subtask to a fresh context | Subagents | `agents` parameter + `allowedTools: ["Agent"]` |
| Coordinate multiple Claude Code instances with shared task lists | Agent teams | Not configured via SDK options — a CLI feature |
| Run deterministic logic on tool calls (audit, block, transform) | Hooks | `hooks` parameter, or shell scripts via `settingSources` |
| Give Claude structured access to an external service | MCP | `mcpServers` parameter |

> "Subagents are ephemeral and isolated: fresh conversation, one task, summary returned to parent. Agent teams coordinate multiple independent Claude Code instances that share a task list and message each other directly. Agent teams are a CLI feature."

Every enabled feature adds to the context window; per-feature costs and layering are covered in the "Extend Claude Code" overview.

## Relationship to concept pages

This summary feeds the SDK-extension concept and comparison pages:

- [SDK setting sources](../concepts/sdk-setting-sources.md) — the `settingSources`/`setting_sources` master switch, per-source locations, and `cwd` behavior
- [SDK project instructions](../concepts/sdk-project-instructions.md) — CLAUDE.md and rules load locations and additivity
- [SDK skills loading](../concepts/sdk-skills-loading.md) — filesystem discovery plus the `skills` option
- [SDK callback hooks](../concepts/sdk-callback-hooks.md) — programmatic vs filesystem hooks in `query()`
- [Statusline / context backup](../concepts/statusline-context-backup.md) — context costs and host-level inputs read regardless of `settingSources`
- [SDK extension features](../comparisons/sdk-extension-features.md) — comparison of CLAUDE.md, skills, subagents, agent teams, hooks, and MCP
