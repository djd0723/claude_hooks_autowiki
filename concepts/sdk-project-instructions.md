---
type: concept
title: "SDK Project Instructions"
created: 2026-06-29
updated: 2026-06-29
tags: [agent-sdk, CLAUDE.md, rules, memory, project-instructions]
source_count: 1
sources:
  - sources/clean/code-claude-com-docs-en-agent-sdk-claude-code-features-md.md
---

# SDK Project Instructions

> `CLAUDE.md` files and `.claude/rules/*.md` files give your agent persistent context about your project: coding conventions, build commands, architecture decisions, and instructions.

When `settingSources` includes `"project"`, the SDK loads these files into context at session start, so the agent follows your project conventions without you repeating them in every prompt. Project instructions are the SDK equivalent of the memory the CLI loads automatically — gated entirely through [SDK Setting Sources](./sdk-setting-sources.md).

## Load locations

| Level | Location | When loaded |
| :---- | :------- | :---------- |
| Project (root) | `<cwd>/CLAUDE.md` or `<cwd>/.claude/CLAUDE.md` | `settingSources` includes `"project"` |
| Project rules | `<cwd>/.claude/rules/*.md` and `.claude/rules/*.md` in every parent directory | `settingSources` includes `"project"` |
| Project (parent dirs) | `CLAUDE.md` files in directories above `cwd` | `settingSources` includes `"project"`, loaded at session start |
| Project (child dirs) | `CLAUDE.md` files in subdirectories of `cwd` | `settingSources` includes `"project"`, loaded **on demand** when the agent reads a file in that subtree |
| Local | `<cwd>/CLAUDE.local.md` and `CLAUDE.local.md` in every parent directory | `settingSources` includes `"local"` |
| User | `~/.claude/CLAUDE.md` | `settingSources` includes `"user"` |
| User rules | `~/.claude/rules/*.md` | `settingSources` includes `"user"` |

## Levels are additive, with no hard precedence

> All levels are additive: if both project and user CLAUDE.md files exist, the agent sees both. There is no hard precedence rule between levels; if instructions conflict, the outcome depends on how Claude interprets them.

Because there is no deterministic override between levels, the source advises writing non-conflicting rules, or stating precedence explicitly in the more specific file — for example, *"These project instructions override any conflicting user-level defaults."*

## CLAUDE.md vs. systemPrompt

You can also inject context directly via `systemPrompt` without using CLAUDE.md files. Reach for CLAUDE.md when you want the **same context shared between interactive Claude Code sessions and your SDK agents**; reach for `systemPrompt` when the context is specific to the SDK process and need not live on disk.

## Related concepts

- [SDK Setting Sources](./sdk-setting-sources.md) — the option that gates whether these files load
- [SDK Skills Loading](./sdk-skills-loading.md) — the on-demand counterpart to always-loaded project instructions
- [SDK Extension Features](../comparisons/sdk-extension-features.md) — when to choose CLAUDE.md over skills, hooks, or subagents
- [Settings Files](./settings-files.md) — the broader set of filesystem settings the SDK can load
