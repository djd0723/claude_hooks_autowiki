---
type: concept
title: "Skill Arguments and Substitutions"
created: 2026-06-29
updated: 2026-06-29
tags: [skills, arguments, substitution, variables, templating]
source_count: 1
sources:
  - sources/clean/code-claude-com-docs-en-skills-md.md
---

# Skill Arguments and Substitutions

Both you and Claude can pass arguments when invoking a [skill](./skills.md). Arguments and a set of built-in variables are available through string substitution in the skill content.

## Passing arguments

`$ARGUMENTS` is replaced with whatever follows the skill name:

```yaml
---
name: fix-issue
description: Fix a GitHub issue
disable-model-invocation: true
---

Fix GitHub issue $ARGUMENTS following our coding standards.
```

Running `/fix-issue 123` makes Claude receive "Fix GitHub issue 123 following our coding standards...". If a skill is invoked with arguments but contains no `$ARGUMENTS`, Claude Code appends `ARGUMENTS: <your input>` to the end of the content so Claude still sees what you typed.

## Substitution reference

| Variable | Description |
| :------- | :---------- |
| `$ARGUMENTS` | All arguments passed when invoking the skill |
| `$ARGUMENTS[N]` | A specific argument by 0-based index, e.g. `$ARGUMENTS[0]` for the first |
| `$N` | Shorthand for `$ARGUMENTS[N]`, e.g. `$0` for the first argument |
| `$name` | A named argument declared in the [`arguments`](./skill-frontmatter.md) frontmatter list. Names map to positions in order |
| `${CLAUDE_SESSION_ID}` | The current session ID — for logging or session-specific files |
| `${CLAUDE_EFFORT}` | The current effort level: `low`, `medium`, `high`, `xhigh`, or `max`. Ultracode reports as `xhigh` |
| `${CLAUDE_SKILL_DIR}` | The directory containing the skill's `SKILL.md`. For plugin skills, the skill's subdirectory, not the plugin root. Use it to reference bundled scripts regardless of working directory |

## Indexed and named arguments

Indexed access by position uses shell-style quoting:

```yaml
Migrate the $ARGUMENTS[0] component from $ARGUMENTS[1] to $ARGUMENTS[2].
```

Running `/migrate-component SearchBar React Vue` fills `SearchBar`, `React`, `Vue`. The `$N` shorthand (`$0`, `$1`, `$2`) is equivalent. Wrap multi-word values in quotes to pass them as one argument: `/my-skill "hello world" second` makes `$0` expand to `hello world` and `$1` to `second`. The `$ARGUMENTS` placeholder always expands to the full string as typed.

Named arguments come from the frontmatter `arguments` list — with `arguments: [issue, branch]`, the placeholder `$issue` expands to the first argument and `$branch` to the second.

## Escaping a literal `$`

To include a literal `$` before a digit, `ARGUMENTS`, or a declared argument name (such as `$1.00` in prose), escape it with a backslash: `\$1.00`. A backslash before any other `$` is left unchanged. Only a single backslash directly before the token escapes it — a doubled backslash like `\\$1` leaves both backslashes in place and `$1` still expands.

`${CLAUDE_SKILL_DIR}` and the other built-in variables are most often used inside [dynamic context injection](./skill-dynamic-context.md) commands to locate bundled scripts.

## Related concepts

- [Skills](./skills.md) — the core concept
- [Skill Frontmatter](./skill-frontmatter.md) — the `arguments` and `argument-hint` fields
- [Skill Dynamic Context](./skill-dynamic-context.md) — running commands whose paths use `${CLAUDE_SKILL_DIR}`
