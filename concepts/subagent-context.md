---
type: concept
title: "Subagent Context"
created: 2026-06-29
updated: 2026-06-29
tags: [subagents, context, memory, nesting, compaction]
source_count: 1
sources:
  - sources/clean/code-claude-com-docs-en-sub-agents-md.md
---

# Subagent Context

Each [subagent](./subagents.md) starts with a fresh, isolated context window. It doesn't see your conversation history, the skills you've already invoked, or the files Claude has already read. Claude composes a delegation message that summarizes the task, and the subagent works from there. The exception is a [fork](./forked-subagents.md), which inherits the parent conversation instead of starting fresh.

## What loads at startup

A non-fork subagent's initial context contains:

- **System prompt** — the agent's own prompt plus environment details Claude Code appends, not the full Claude Code system prompt. Custom subagents define theirs in the [markdown body](./subagent-configuration.md) or `prompt` field; built-in agents have predefined prompts.
- **Task message** — the delegation prompt Claude writes when it hands off the work.
- **CLAUDE.md and memory** — every level of the memory hierarchy the main conversation loads, including `~/.claude/CLAUDE.md`, project rules, `CLAUDE.local.md`, and managed policy files.
- **Git status** — a snapshot taken at the start of the parent session. Absent when the working directory isn't a Git repository or when `includeGitInstructions` is `false`.
- **Preloaded skills** — full content of any skill named in the agent's `skills` field.

**Explore and Plan are the only subagents that omit CLAUDE.md and git status**, to keep research fast and inexpensive. There is no frontmatter field or per-agent setting to change which agents skip them. The main conversation reads Explore and Plan results with full CLAUDE.md context, so most rules don't need to reach the subagent itself; if a rule must (such as "ignore the `vendor/` directory"), restate it in the delegation prompt.

## Resume subagents

Each subagent invocation creates a new instance with fresh context. To continue an existing subagent's work, ask Claude to resume it — resumed subagents retain their full conversation history, including all previous tool calls, results, and reasoning, picking up exactly where they stopped.

When a subagent completes, Claude receives its agent ID and resumes it via the `SendMessage` tool (always available for resuming by ID or name). If a stopped subagent receives a `SendMessage`, it auto-resumes in the background without a new Agent invocation. The built-in **Explore and Plan agents are one-shot and return no agent ID**, so they can't be resumed; use `general-purpose` or a custom subagent when you need to continue work.

Agent IDs can be found in transcript files at `~/.claude/projects/{project}/{sessionId}/subagents/`, stored as `agent-{agentId}.jsonl`. Subagent transcripts persist independently of the main conversation:

- **Main conversation compaction** leaves subagent transcripts unaffected — they're stored in separate files.
- **Session persistence**: transcripts persist within their session, so you can resume a subagent after restarting Claude Code by resuming the same session.
- **Automatic cleanup** is governed by `cleanupPeriodDays`, which defaults to 30 days.

## Auto-compaction

Subagents support automatic compaction using the same logic as the main conversation, triggered under the same conditions; `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` applies to subagents as well. Compaction events are logged in subagent transcript files as a `compact_boundary` system entry whose `preTokens` value shows how many tokens were used before compaction occurred.

## Spawn nested subagents

As of Claude Code v2.1.172, a subagent can spawn its own subagents. Use this when a delegated task itself splits into parallel subtasks, such as a reviewer subagent that dispatches a verifier per finding — the intermediate output never reaches your main conversation, and only the top-level subagent's summary returns to you.

A nested subagent is configured the same way as a top-level one and resolves from the same [scopes](./subagent-configuration.md). **Depth** is counted as the number of subagent levels below the main conversation, regardless of [foreground or background](./subagent-invocation.md) execution. A subagent at depth five doesn't receive the Agent tool and can't spawn further; the limit is fixed and not configurable. As of v2.1.187, a background subagent's depth is fixed when first spawned, and resuming it later doesn't change that depth.

To prevent a specific subagent from spawning others, omit `Agent` from its [`tools`](./subagent-tool-access.md) list or add it to `disallowedTools`.

## Related concepts

- [Subagents](./subagents.md) — the core concept
- [Subagent Configuration](./subagent-configuration.md) — `skills`, `memory`, and the body that becomes the system prompt
- [Subagent Invocation](./subagent-invocation.md) — foreground vs background, which affects depth counting
- [Forked Subagents](./forked-subagents.md) — the one subagent type that inherits the full conversation
