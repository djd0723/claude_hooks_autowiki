---
type: concept
title: "SDK Skills Loading"
created: 2026-06-29
updated: 2026-06-29
tags: [agent-sdk, skills, settingSources, on-demand, allowedTools]
source_count: 1
sources:
  - sources/clean/code-claude-com-docs-en-agent-sdk-claude-code-features-md.md
---

# SDK Skills Loading

> Skills are markdown files that give your agent specialized knowledge and invocable workflows. Unlike `CLAUDE.md` (which loads every session), skills load on demand. The agent receives skill descriptions at startup and loads the full content when relevant.

In the SDK, skills are **discovered from the filesystem through `settingSources`** and then gated by a second `skills` option that controls which discovered skills are actually enabled.

## Discovery and the `skills` option

Skills are discovered through [SDK Setting Sources](./sdk-setting-sources.md) — e.g. `.claude/skills/` is found when `settingSources` includes `"project"`. From there:

- When the `skills` option on `query()` is **omitted**, discovered user and project skills are enabled and the Skill tool is available — matching CLI behavior.
- To control which skills are enabled, pass `skills` as `"all"`, a list of skill names, or `[]` to disable all.

## The Skill tool and `allowedTools`

When `skills` is set, the SDK adds the Skill tool to `allowedTools` automatically. But there is a sharp edge:

> If you also pass an explicit `tools` list, include `"Skill"` in that list so Claude can invoke skills.

So the automatic injection only helps when you are not overriding the tool list yourself.

## No programmatic registration

> Skills must be created as filesystem artifacts (`.claude/skills/<name>/SKILL.md`). The SDK does not have a programmatic API for registering skills.

This is the key asymmetry with [SDK Callback Hooks](./sdk-callback-hooks.md): hooks can be passed as in-process callbacks, but skills must exist on disk and be reached through `settingSources`.

## Related concepts

- [SDK Setting Sources](./sdk-setting-sources.md) — how skills are discovered from the filesystem
- [Skills](./skills.md) — what a skill is and how it is authored
- [Skill Discovery](./skill-discovery.md) — where skills live, scope/precedence, and live change detection
- [SDK Extension Features](../comparisons/sdk-extension-features.md) — skills vs. CLAUDE.md vs. subagents vs. hooks
