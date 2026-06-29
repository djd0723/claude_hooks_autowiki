---
type: overview
created: 2026-06-29
updated: 2026-06-29
tags: [hooks, orientation]
---

# Claude Code Hooks — Overview

Start-here orientation: what this wiki covers, its major themes, and where to
begin. Maintained by the indexer; evolves as the wiki grows.

## What this wiki covers

Claude Code **hooks** — user-defined shell commands, HTTP endpoints, LLM prompts,
and subagents that run automatically at fixed lifecycle points. The wiki is built
from the official Claude Code hooks documentation (guide + reference).

## Core thesis

Hooks exist to solve a fundamental reliability problem: you cannot guarantee an
LLM will always take an action. Hooks provide the **deterministic layer** that
sits alongside the model — running regardless of what the model decides.

> "They provide deterministic control over Claude Code's behavior, ensuring
> certain actions always happen rather than relying on the LLM to choose to
> run them."

## Major themes

- **30+ lifecycle events** — from `SessionStart` to `PostToolUse` to `Stop` to `FileChanged`
- **Five hook types** — command (default), http, mcp_tool, prompt, agent
- **Two-tier filtering** — `matcher` (group-level, by event/tool name) and `if` (hook-level, by arguments)
- **Structured decision control** — each event has its own blocking mechanism; `PreToolUse` alone has four outcomes (allow/deny/ask/defer)
- **Scope and permission interaction** — hooks fire before permission-mode checks; a hook `deny` blocks even in `bypassPermissions` mode

## Where to start

| Goal | Go here |
| :--- | :------ |
| Set up a hook for the first time | [Hooks Guide summary](summaries/hooks-guide.md) |
| Look up a specific event | [Hook Lifecycle Events](concepts/hook-lifecycle-events.md) |
| Choose between hook types | [Hook Types](concepts/hook-types.md) |
| Understand blocking/allow/deny | [Hook Decision Control](concepts/hook-decision-control.md) |
| Full JSON schemas | [Hook Input and Output](concepts/hook-input-output.md) |
| See the full thesis | [Synthesis](synthesis.md) |
