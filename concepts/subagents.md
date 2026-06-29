---
type: concept
title: "Subagents"
created: 2026-06-29
updated: 2026-06-29
tags: [subagents, delegation, context, agents]
source_count: 1
sources:
  - sources/clean/code-claude-com-docs-en-sub-agents-md.md
---

# Subagents

> Subagents are specialized AI assistants that handle specific types of tasks.

A subagent runs in **its own context window** with a custom system prompt, specific tool access, and independent permissions. When Claude encounters a task that matches a subagent's `description`, it delegates to that subagent, which works independently and returns only its results to the main conversation.

Use a subagent when a side task would flood your main conversation with search results, logs, or file contents you won't reference again: the subagent does that work in its own context and returns only the summary. Define a *custom* subagent when you keep spawning the same kind of worker with the same instructions.

## What subagents give you

- **Preserve context** by keeping exploration and implementation out of your main conversation
- **Enforce constraints** by limiting which [tools](./subagent-tool-access.md) a subagent can use
- **Reuse configurations** across projects with user-level subagents
- **Specialize behavior** with focused system prompts for specific domains
- **Control costs** by routing tasks to faster, cheaper models like Haiku

## Built-in subagents

Claude Code includes built-in subagents it uses automatically when appropriate. Each inherits the parent conversation's permissions with additional tool restrictions.

| Subagent | Model | Tools | Purpose |
| :------- | :---- | :---- | :------ |
| **Explore** | Haiku | read-only (Write/Edit denied) | file discovery, code search, codebase exploration |
| **Plan** | inherits | read-only (Write/Edit denied) | codebase research during plan mode |
| **general-purpose** | inherits | all tools | complex, multi-step tasks needing both exploration and action |
| `statusline-setup` | Sonnet | — | configures your status line when you run `/statusline` |
| `claude-code-guide` | Haiku | — | answers questions about Claude Code features |

When invoking Explore, Claude specifies a thoroughness level: **quick** for targeted lookups, **medium** for balanced exploration, or **very thorough** for comprehensive analysis.

Built-in subagents are always registered in interactive sessions. To restrict them, deny a specific type via `permissions.deny` or deny the `Agent` tool entirely (see [Subagent Tool Access](./subagent-tool-access.md)). In non-interactive (headless) mode and the Agent SDK, set `CLAUDE_AGENT_SDK_DISABLE_BUILTIN_AGENTS=1` to remove all built-in types and supply only your own.

## When to use a subagent

Use **subagents** when:

- The task produces verbose output you don't need in your main context
- You want to enforce specific tool restrictions or permissions
- The work is self-contained and can return a summary

Use the **main conversation** when the task needs frequent back-and-forth, multiple phases share significant context, you're making a quick targeted change, or latency matters (subagents start fresh and may need time to gather context).

> Consider Skills instead when you want reusable prompts or workflows that run in the main conversation context rather than isolated subagent context.

For a quick question about something already in your conversation, use `/btw` instead of a subagent: it sees your full context but has no tool access, and the answer is discarded rather than added to history.

## Related concepts

- [Subagent Configuration](./subagent-configuration.md) — file format, frontmatter fields, scopes, and model selection
- [Subagent Tool Access](./subagent-tool-access.md) — tools, MCP scoping, permission modes, and spawn restrictions
- [Subagent Invocation](./subagent-invocation.md) — automatic delegation and explicit invocation
- [Subagent Context](./subagent-context.md) — what loads at startup, resume, and nested subagents
- [Subagent Hooks](./subagent-hooks.md) — lifecycle hooks scoped to a subagent
- [Forked Subagents](./forked-subagents.md) — subagents that inherit the full conversation
