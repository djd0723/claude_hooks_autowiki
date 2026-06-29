---
type: summary
title: "Hooks Guide — Automate actions with hooks"
slug: code-claude-com-docs-en-hooks-guide-md
created: 2026-06-29
updated: 2026-06-29
tags: [hooks, automation, lifecycle, configuration]
sources:
  - sources/clean/code-claude-com-docs-en-hooks-guide-md.md
---

# Summary: Automate actions with hooks

Official Claude Code guide covering how to set up and use hooks — user-defined shell commands that run automatically at lifecycle points.

## What this source covers

- How hooks work: events, matchers, input/output/exit codes
- The 30+ [hook lifecycle events](../concepts/hook-lifecycle-events.md) and when they fire
- Five [hook types](../concepts/hook-types.md): command, http, mcp_tool, prompt, agent
- [Matcher patterns](../concepts/hook-matchers.md) and the `if` field for argument-level filtering
- [Exit codes and JSON structured output](../concepts/hook-exit-codes.md) for communicating decisions
- [Hook scope and location](../concepts/hook-scope.md): user, project, plugin, skill
- Common recipes: notifications, auto-format, file protection, context re-injection, audit logging, env reload, permission auto-approval
- Limitations, permission mode interactions, and debug techniques

## Key thesis from source

> "Hooks are user-defined shell commands that execute at specific points in Claude Code's lifecycle. They provide deterministic control over Claude Code's behavior, ensuring certain actions always happen rather than relying on the LLM to choose to run them."

## Scope

This is the **tutorial guide**, not the full reference. For full event schemas, JSON formats, and advanced features, the source points to the Hooks reference (`/en/hooks`).
