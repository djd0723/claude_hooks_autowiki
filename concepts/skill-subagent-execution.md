---
type: concept
title: "Skill Subagent Execution"
created: 2026-06-29
updated: 2026-06-29
tags: [skills, subagents, fork, context, agent]
source_count: 1
sources:
  - sources/clean/code-claude-com-docs-en-skills-md.md
---

# Skill Subagent Execution

Add `context: fork` to a [skill's](./skills.md) frontmatter to run it in isolation. The skill content becomes the prompt that drives a [subagent](./subagents.md), and it does not have access to your conversation history.

> "`context: fork` only makes sense for skills with explicit instructions. If your skill contains guidelines like 'use these API conventions' without a task, the subagent receives the guidelines but no actionable prompt, and returns without meaningful output."

This pairs with the two [content types](./skills.md): only **task content** makes sense to fork; pure **reference content** should run inline.

## Two directions of skill ↔ subagent integration

Skills and subagents combine in opposite directions:

| Approach | System prompt | Task | Also loads |
| :------- | :------------ | :--- | :--------- |
| Skill with `context: fork` | From agent type | `SKILL.md` content | CLAUDE.md, except when the agent is Explore or Plan |
| Subagent with `skills` field | Subagent's markdown body | Claude's delegation message | Preloaded skills + CLAUDE.md |

With `context: fork`, you write the task in your skill and pick an agent type to execute it. The built-in Explore and Plan agents [skip CLAUDE.md and git status](./subagent-context.md) to keep their context small, so a forked skill using `agent: Explore` sees only the `SKILL.md` content and the agent's own system prompt.

For the inverse — a custom subagent that loads skills as reference material — see the [`skills` frontmatter field](./subagent-configuration.md), which injects the full skill content at startup (unlike the on-demand loading of a [regular session](./skill-content-lifecycle.md)). A skill with `disable-model-invocation: true` is excluded from this preloading.

## How a forked skill runs

```yaml
---
name: deep-research
description: Research a topic thoroughly
context: fork
agent: Explore
---

Research $ARGUMENTS thoroughly:

1. Find relevant files using Glob and Grep
2. Read and analyze the code
3. Summarize findings with specific file references
```

1. A new isolated context is created.
2. The subagent receives the skill content as its prompt.
3. The `agent` field determines the execution environment (model, tools, permissions).
4. Results are summarized and returned to your main conversation.

The `agent` field specifies which subagent configuration to use — a built-in agent (`Explore`, `Plan`, `general-purpose`) or any custom subagent from `.claude/agents/`. If omitted, it uses `general-purpose`.

A forked skill is distinct from a [`/fork` subagent](./forked-subagents.md): a `/fork` inherits the *full* conversation, whereas a forked skill starts from an isolated context seeded only by the skill content.

## Related concepts

- [Skills](./skills.md) — the core concept
- [Skill Frontmatter](./skill-frontmatter.md) — the `context`, `agent`, `model`, and `effort` fields
- [Subagents](./subagents.md) — the agents a forked skill runs in
- [Subagent Configuration](./subagent-configuration.md) — the `skills` field that preloads skills into a subagent
- [Forked Subagents](./forked-subagents.md) — `/fork`, which inherits the full conversation instead
- [Skill Content Lifecycle](./skill-content-lifecycle.md) — on-demand vs. preloaded skill content
