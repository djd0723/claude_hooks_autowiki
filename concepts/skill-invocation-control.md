---
type: concept
title: "Skill Invocation Control"
created: 2026-06-29
updated: 2026-06-29
tags: [skills, invocation, permissions, allowed-tools, skilloverrides, visibility]
source_count: 1
sources:
  - sources/clean/code-claude-com-docs-en-skills-md.md
---

# Skill Invocation Control

By default, both you and Claude can invoke any [skill](./skills.md): you type `/skill-name`, and Claude loads it automatically when relevant. Two [frontmatter](./skill-frontmatter.md) fields restrict this:

- **`disable-model-invocation: true`** — only *you* can invoke the skill. Use for workflows with side effects or timing you want to control, like `/commit`, `/deploy`, or `/send-slack-message`. "You don't want Claude deciding to deploy because your code looks ready."
- **`user-invocable: false`** — only *Claude* can invoke the skill. Use for background knowledge that isn't a meaningful user action, like a `legacy-system-context` skill that explains how an old system works.

These two fields also change when content loads into context:

| Frontmatter | You can invoke | Claude can invoke | When loaded into context |
| :---------- | :------------- | :---------------- | :----------------------- |
| (default) | Yes | Yes | Description always in context, full skill loads when invoked |
| `disable-model-invocation: true` | Yes | No | Description not in context, full skill loads when you invoke |
| `user-invocable: false` | No | Yes | Description always in context, full skill loads when invoked |

In a regular session, descriptions load so Claude knows what's available; full content loads only on invocation. [Subagents with preloaded skills](./skill-subagent-execution.md) differ — the full content is injected at startup.

## Pre-approve tools for a skill

The `allowed-tools` field grants permission for the listed tools while the skill is active, so Claude uses them without prompting:

> "It does not restrict which tools are available: every tool remains callable, and your permission settings still govern tools that are not listed."

For skills checked into a project's `.claude/skills/`, `allowed-tools` takes effect after you accept the workspace trust dialog — review project skills before trusting a repo, since a skill can grant itself broad tool access. Example:

```yaml
---
name: commit
description: Stage and commit the current changes
disable-model-invocation: true
allowed-tools: Bash(git add *) Bash(git commit *) Bash(git status *)
---
```

To *remove* tools from the pool while a skill is active, list them in `disallowed-tools`; the restriction clears on your next message. To block tools across all skills and prompts, add deny rules in your [permission settings](./permission-settings.md).

## Restrict which skills Claude can invoke

By default Claude can invoke any skill without `disable-model-invocation: true`. A few built-in commands (`/init`, `/review`, `/security-review`) are also available through the Skill tool; others like `/compact` are not. Three controls:

- **Disable all skills** — deny the `Skill` tool in `/permissions`.
- **Allow or deny specific skills** — use [permission rules](./permission-settings.md): `Skill(name)` for exact match, `Skill(name *)` for prefix match with any arguments. E.g. `Skill(commit)`, `Skill(deploy *)` in deny rules.
- **Hide individual skills** — add `disable-model-invocation: true` to frontmatter, removing the skill from Claude's context entirely.

> "The `user-invocable` field only controls menu visibility, not Skill tool access. Use `disable-model-invocation: true` to block programmatic invocation."

## Override visibility from settings — `skillOverrides`

The `skillOverrides` setting controls visibility from your [settings](./settings-files.md) instead of the skill's own frontmatter — useful for skills you don't want to edit (shared repos, MCP-provided). The `/skills` menu writes it for you (`Space` cycles states, `Enter` saves to `.claude/settings.local.json`). Each key is a skill name; each value one of four states:

| Value | Listed to Claude | In `/` menu |
| :---- | :--------------- | :---------- |
| `"on"` | Name and description | Yes |
| `"name-only"` | Name only | Yes |
| `"user-invocable-only"` | Hidden | Yes |
| `"off"` | Hidden | Hidden |

A skill absent from `skillOverrides` is treated as `"on"`. Plugin skills are not affected — manage those through `/plugin`.

## When descriptions get cut short

All skill *names* are always listed, but descriptions are shortened to fit a character budget (1% of the model's context window by default). When it overflows, descriptions for the least-invoked skills drop first. Run `/doctor` to see how many are shortened or dropped. To raise the budget, set `skillListingBudgetFraction` (e.g. `0.02`) or `SLASH_COMMAND_TOOL_CHAR_BUDGET`; to free budget, set low-priority entries to `"name-only"`. Each entry's combined `description` + `when_to_use` is capped at 1,536 characters regardless of budget (configurable via `maxSkillDescriptionChars`).

## Related concepts

- [Skills](./skills.md) — the core concept
- [Skill Frontmatter](./skill-frontmatter.md) — the fields referenced here
- [Skill Content Lifecycle](./skill-content-lifecycle.md) — what "loads into context" means over a session
- [Permission Settings](./permission-settings.md) — the `Skill(...)` rule syntax and deny rules
