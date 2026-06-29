---
type: concept
title: "Skill Frontmatter"
created: 2026-06-29
updated: 2026-06-29
tags: [skills, frontmatter, configuration, command-name, reference]
source_count: 1
sources:
  - sources/clean/code-claude-com-docs-en-skills-md.md
---

# Skill Frontmatter

A [skill](./skills.md) is configured through YAML frontmatter at the top of `SKILL.md`, followed by the markdown content:

```yaml
---
name: my-skill
description: What this skill does
disable-model-invocation: true
allowed-tools: Read Grep
---

Your skill instructions here...
```

> "All fields are optional. Only `description` is recommended so Claude knows when to use the skill."

## Field reference

| Field | Required | Description |
| :---- | :------- | :---------- |
| `name` | No | Display name shown in skill listings. Defaults to the directory name. Usually does **not** change the command you type |
| `description` | Recommended | What the skill does and when to use it. Claude uses this to decide when to apply the skill. If omitted, uses the first paragraph of content. Put the key use case first — combined `description` + `when_to_use` is truncated at 1,536 characters in the listing |
| `when_to_use` | No | Extra context for when to invoke, such as trigger phrases. Appended to `description`; counts toward the 1,536-character cap |
| `argument-hint` | No | Hint shown during autocomplete, e.g. `[issue-number]` or `[filename] [format]` |
| `arguments` | No | Named positional [arguments](./skill-arguments.md) for `$name` substitution. Space-separated string or YAML list |
| `disable-model-invocation` | No | `true` prevents Claude from auto-loading the skill (manual `/name` only). Also prevents [preloading into subagents](./skill-subagent-execution.md). Default `false` |
| `user-invocable` | No | `false` hides the skill from the `/` menu (background knowledge only). Default `true` |
| `allowed-tools` | No | Tools Claude may use without asking permission while the skill is active. Space/comma string or YAML list |
| `disallowed-tools` | No | Tools removed from the pool while the skill is active. Clears on your next message |
| `model` | No | Model while the skill is active. Same values as `/model`, or `inherit`. Applies for the rest of the current turn only |
| `effort` | No | Effort level while active: `low`, `medium`, `high`, `xhigh`, `max`. Overrides session effort |
| `context` | No | Set to `fork` to run in a [forked subagent context](./skill-subagent-execution.md) |
| `agent` | No | Which subagent type to use when `context: fork` is set |
| `hooks` | No | [Hooks](./hook-scope.md) scoped to this skill's lifecycle |
| `paths` | No | Glob patterns that limit when the skill auto-activates — see [Path-Scoped Skills](./path-scoped-skills.md). Comma string or YAML list |
| `shell` | No | Shell for `` !`command` `` and `` ```! `` blocks: `bash` (default) or `powershell` |

Tool-related fields (`allowed-tools`, `disallowed-tools`) are detailed in [Skill Invocation Control](./skill-invocation-control.md); `model`, `effort`, `context`, and `agent` interact with [Skill Subagent Execution](./skill-subagent-execution.md).

## How a skill gets its command name

The command you type comes from **where the skill file lives**, not the `name` field. The frontmatter `name` sets the display label in listings and — except for a plugin-root `SKILL.md` — does not change what you type after `/`.

| Skill location | Command name source | Example |
| :------------- | :------------------ | :------ |
| `~/.claude/skills/` or `.claude/skills/` directory | Directory name | `.claude/skills/deploy-staging/SKILL.md` → `/deploy-staging` |
| [Nested](./skill-discovery.md) `.claude/skills/`, when the name clashes | Subdirectory path relative to working dir, then skill directory name | `apps/web/.claude/skills/deploy/SKILL.md` → `/apps/web:deploy` |
| File under `.claude/commands/` | File name without extension | `.claude/commands/deploy.md` → `/deploy` |
| Plugin `skills/` subdirectory | Directory name, namespaced by plugin | `my-plugin/skills/review/SKILL.md` → `/my-plugin:review` |
| Plugin root `SKILL.md` | Frontmatter `name`, plugin directory name as fallback | `my-plugin/SKILL.md` with `name: review` → `/my-plugin:review` |

The plugin-root case is the one place where `name` sets the command name, because there is no skill directory to take it from. See [Plugin Components](./plugin-components.md) for plugin-shipped skills.

## Related concepts

- [Skills](./skills.md) — the core concept this configures
- [Skill Arguments](./skill-arguments.md) — the `arguments` field and `$` substitutions
- [Skill Invocation Control](./skill-invocation-control.md) — `disable-model-invocation`, `user-invocable`, `allowed-tools`
- [Skill Discovery](./skill-discovery.md) — how location determines scope and command name
- [Path-Scoped Skills](./path-scoped-skills.md) — the practitioner view of the `paths:` field: scoping strategies and anti-patterns
- [Tool Permission Rules](./tool-permission-rules.md) — the `ToolName(specifier)` format the `allowed-tools` field uses
