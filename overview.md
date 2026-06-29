---
type: overview
created: 2026-06-29
updated: 2026-06-29
tags: [hooks, plugins, orientation]
---

# Claude Code Hooks & Plugins — Overview

Start-here orientation: what this wiki covers, its major themes, and where to
begin. Maintained by the indexer; evolves as the wiki grows.

## What this wiki covers

Claude Code **hooks** and **plugins** — the deterministic automation and distribution layers
that sit alongside the Claude model. Hooks are user-defined shell commands, HTTP endpoints,
LLM prompts, and subagents that fire automatically at lifecycle points. Plugins bundle hooks
and other components (skills, agents, MCP servers, LSP servers, monitors, themes) into
installable, versioned units.

The wiki is built from the official Claude Code hooks and plugins documentation.

## Core thesis

Hooks solve a fundamental reliability problem: you cannot guarantee an LLM will always take
an action. Hooks provide the **deterministic layer** that runs regardless of what the model
decides.

> "They provide deterministic control over Claude Code's behavior, ensuring
> certain actions always happen rather than relying on the LLM to choose to
> run them."

**Plugins are the distribution layer above hooks.** A plugin packages hooks alongside other
components as a single installable unit. Plugin hooks are mechanically identical to
user-defined hooks — same events, same exit code protocol, same JSON I/O — but travel with a
versioned, scoped bundle.

## Major themes

**Hooks**
- **30+ lifecycle events** — from `SessionStart` to `PostToolUse` to `Stop` to `FileChanged`
- **Five hook types** — command (default), http, mcp_tool, prompt, agent
- **Two-tier filtering** — `matcher` (group-level, by event/tool name) and `if` (hook-level, by arguments)
- **Structured decision control** — each event has its own blocking mechanism; `PreToolUse` alone has four outcomes (allow/deny/ask/defer)
- **Scope and permission interaction** — hooks fire before permission-mode checks; a hook `deny` blocks even in `bypassPermissions` mode

**Plugins**
- **Seven component types** — skills, agents, hooks, MCP servers, LSP servers, monitors, themes
- **Four installation scopes** — user (personal), project (team/VC), local (gitignored), managed (admin-controlled)
- **Skills-directory plugins** — discovered in place, no install step; scaffold with `claude plugin init`
- **Monitors** — persistent background processes delivering stdout as real-time notifications (v2.1.105+)
- **Manifest-optional** — Claude Code auto-discovers components; only `name` is required if a manifest is present

## Where to start

| Goal | Go here |
| :--- | :------ |
| Set up a hook for the first time | [Hooks Guide summary](summaries/hooks-guide.md) |
| Look up a specific event | [Hook Lifecycle Events](concepts/hook-lifecycle-events.md) |
| Choose between hook types | [Hook Types](concepts/hook-types.md) |
| Understand blocking/allow/deny | [Hook Decision Control](concepts/hook-decision-control.md) |
| Full JSON schemas | [Hook Input and Output](concepts/hook-input-output.md) |
| Build or install a plugin | [Plugin Components](concepts/plugin-components.md) |
| Understand plugin scopes | [Plugin Installation Scopes](concepts/plugin-installation-scopes.md) |
| See the full thesis | [Synthesis](synthesis.md) |
