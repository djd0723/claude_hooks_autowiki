---
type: concept
title: "Forked Subagents"
created: 2026-06-29
updated: 2026-06-29
tags: [subagents, fork, context, prompt-cache]
source_count: 1
sources:
  - sources/clean/code-claude-com-docs-en-sub-agents-md.md
---

# Forked Subagents

A **fork** is a [subagent](./subagents.md) that inherits the entire conversation so far instead of starting fresh. This drops the input isolation that subagents otherwise provide: a fork sees the same system prompt, tools, model, and message history as the main session, so you can hand it a side task without re-explaining the situation. The fork's own tool calls still stay out of your conversation and only its final result comes back, so your main [context window](./subagent-context.md) stays clean.

Use a fork when a named subagent would need too much background to be useful, or when you want to try several approaches in parallel from the same starting point.

> Forked subagents require Claude Code v2.1.117 or later. From v2.1.161 the `/fork` command is enabled by default; on earlier versions it requires setting `CLAUDE_CODE_FORK_SUBAGENT` to `1`. Letting Claude itself spawn forks is experimental and may change.

## Enabling fork mode

Set `CLAUDE_CODE_FORK_SUBAGENT` to `1` to enable explicitly or `0` to disable, honored in interactive mode and via the SDK or `claude -p`. Enabling fork mode changes Claude Code in two ways:

- Claude can spawn a fork by requesting the `fork` subagent type explicitly. Spawns without a subagent type still use general-purpose, and named subagents such as Explore still spawn as before.
- Every subagent spawn runs in the [background](./subagent-invocation.md), whether a fork or a named subagent. Set `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS=1` to keep spawns synchronous.

## Starting a fork

Start one yourself with `/fork` followed by a directive — Claude Code names the fork from the first words of the directive:

```text
/fork draft unit tests for the parser changes so far
```

The fork appears in a panel below your prompt and runs in the background while you keep working; when it finishes, its result arrives as a message in your main conversation. In the panel, `↑`/`↓` move between rows, `Enter` opens the selected fork's transcript to send follow-up messages, `x` dismisses a finished fork or stops a running one, and `Esc` returns focus to the prompt input.

## How forks differ from named subagents

A fork inherits everything the main session has at the moment it spawns; a named subagent starts from its own definition.

| | Fork | Named subagent |
| :--- | :--- | :--- |
| Context | Full conversation history | Fresh context with the prompt you pass |
| System prompt and tools | Same as main session | From the subagent's [definition file](./subagent-configuration.md) |
| Model | Same as main session | From the subagent's `model` field |
| Permissions | Prompts surface in your terminal | Prompts surface in your main session when running in the background |
| Prompt cache | Shared with main session | Separate cache |

Because a fork's system prompt and tool definitions are identical to the parent, its first request reuses the parent's prompt cache, making forking cheaper than spawning a fresh subagent for tasks that need the same context. When Claude spawns a fork through the Agent tool, it can pass `isolation: "worktree"` so the fork's file edits are written to a separate git worktree instead of your checkout.

## Limitations

`CLAUDE_CODE_FORK_SUBAGENT=1` enables fork mode in interactive mode, non-interactive (headless) mode, and the Agent SDK; setting it to `0` disables fork mode everywhere, including any server-side rollout. **A fork can't spawn another fork.** It can spawn other subagent types, and those count toward the [depth limit](./subagent-context.md).

## Related concepts

- [Subagents](./subagents.md) — the core concept a fork specializes
- [Subagent Context](./subagent-context.md) — the fresh-context model a fork opts out of, and the depth limit
- [Subagent Invocation](./subagent-invocation.md) — background execution that fork mode forces on every spawn
- [Large-Codebase Playbook](./large-codebase-playbook.md) — the exploration-vs-editing split (Strategy 7) that read-only sub-agents enable
