---
type: comparison
title: "Choosing an SDK extension feature"
created: 2026-06-29
updated: 2026-06-29
tags: [agent-sdk, extension, skills, hooks, subagents, mcp, decision]
sources:
  - sources/clean/code-claude-com-docs-en-agent-sdk-claude-code-features-md.md
---

# Choosing an SDK extension feature

The Agent SDK exposes several ways to extend an agent's behavior — project instructions, skills, subagents, hooks, MCP, and (at the CLI level) agent teams. They overlap enough that the real question is *which surface fits a given goal.* This page maps goals to features and to the concrete SDK option you reach for.

## Goal → feature → SDK surface

| You want to... | Use | SDK surface |
| :------------- | :-- | :---------- |
| Set project conventions your agent always follows | CLAUDE.md | `settingSources: ["project"]` loads it automatically |
| Give the agent reference material it loads when relevant | Skills | `settingSources` + `skills` option |
| Run a reusable workflow (deploy, review, release) | User-invocable skills | `settingSources` + `skills` option |
| Delegate an isolated subtask to a fresh context (research, review) | Subagents | `agents` parameter + `allowedTools: ["Agent"]` |
| Coordinate multiple Claude Code instances with shared task lists and direct messaging | Agent teams | Not configured via SDK options — a CLI feature where one session is the team lead |
| Run deterministic logic on tool calls (audit, block, transform) | Hooks | `hooks` parameter with callbacks, or shell scripts loaded via `settingSources` |
| Give Claude structured tool access to an external service | MCP | `mcpServers` parameter |

## Always-on vs. on-demand

The first axis to decide on is **when** the context loads:

- **CLAUDE.md** loads every session — use it for conventions the agent should *always* follow. See [SDK Project Instructions](../concepts/sdk-project-instructions.md).
- **Skills** load on demand — the agent sees descriptions at startup and pulls full content only when relevant. See [SDK Skills Loading](../concepts/sdk-skills-loading.md).

Every feature you enable adds to the agent's context window, so the on-demand model exists precisely to keep that cost down.

## Subagents vs. agent teams

> Subagents are ephemeral and isolated: fresh conversation, one task, summary returned to parent. Agent teams coordinate multiple independent Claude Code instances that share a task list and message each other directly. Agent teams are a CLI feature.

Subagents are the SDK-native isolation primitive (`agents` + the `Agent` tool); agent teams are a CLI-level coordination feature with no direct SDK option. See [Subagents](../concepts/subagents.md).

## Deterministic control: hooks

For audit/block/transform logic on tool calls, hooks come in two interoperating flavours — filesystem hooks loaded via `settingSources` and in-process callbacks passed to `query()`. See [SDK Callback Hooks](../concepts/sdk-callback-hooks.md) for the programmatic side.

## Related

- [SDK Setting Sources](../concepts/sdk-setting-sources.md) — the option behind CLAUDE.md, skills, and filesystem hooks
- [SDK Project Instructions](../concepts/sdk-project-instructions.md) — the always-on option
- [SDK Skills Loading](../concepts/sdk-skills-loading.md) — the on-demand option
- [SDK Callback Hooks](../concepts/sdk-callback-hooks.md) — deterministic tool-call control
- [Subagents](../concepts/subagents.md) — isolated delegation
