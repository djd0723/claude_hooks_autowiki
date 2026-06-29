---
type: concept
title: "Skill Content Lifecycle"
created: 2026-06-29
updated: 2026-06-29
tags: [skills, context, lifecycle, compaction, tokens]
source_count: 1
sources:
  - sources/clean/code-claude-com-docs-en-skills-md.md
---

# Skill Content Lifecycle

When you or Claude invoke a [skill](./skills.md), its rendered `SKILL.md` content enters the conversation as a single message and stays there for the rest of the session:

> "Claude Code does not re-read the skill file on later turns, so write guidance that should apply throughout a task as standing instructions rather than one-time steps."

This is why skill content is a recurring token cost and why the body should stay concise — every line persists across turns.

## Survival through auto-compaction

[Auto-compaction](./subagent-context.md) carries invoked skills forward within a token budget. When the conversation is summarized to free context, Claude Code re-attaches the most recent invocation of each skill *after* the summary, keeping:

- the **first 5,000 tokens** of each skill, and
- a **combined budget of 25,000 tokens** across all re-attached skills.

The budget fills starting from the most recently invoked skill, so older skills can be dropped entirely after compaction if you invoked many in one session.

## When a skill seems to stop working

> "If a skill seems to stop influencing behavior after the first response, the content is usually still present and the model is choosing other tools or approaches."

Remedies:

- Strengthen the skill's `description` and instructions so the model keeps preferring it.
- Use [hooks](./hook-types.md) to enforce behavior deterministically rather than relying on the model.
- If the skill is large or you invoked several others after it, re-invoke it after compaction to restore the full content.

## Contrast: preloaded subagent skills

The lifecycle above describes a regular session, where a skill's *description* sits in context and the full body loads on invocation. A [subagent that preloads skills](./skill-subagent-execution.md) works differently — the full skill content is injected at the subagent's startup instead of on demand.

## Related concepts

- [Skills](./skills.md) — the core concept
- [Skill Invocation Control](./skill-invocation-control.md) — when descriptions vs. full content load
- [Skill Subagent Execution](./skill-subagent-execution.md) — the preloaded-skills alternative
- [Subagent Context](./subagent-context.md) — the auto-compaction mechanism shared with subagents
