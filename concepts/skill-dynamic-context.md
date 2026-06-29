---
type: concept
title: "Skill Dynamic Context Injection"
created: 2026-06-29
updated: 2026-06-29
tags: [skills, dynamic-context, shell, preprocessing, injection]
source_count: 1
sources:
  - sources/clean/code-claude-com-docs-en-skills-md.md
---

# Skill Dynamic Context Injection

The `` !`<command>` `` syntax runs shell commands **before** the [skill](./skills.md) content is sent to Claude. The command output replaces the placeholder, so Claude receives actual data, not the command itself.

```yaml
---
name: pr-summary
description: Summarize changes in a pull request
context: fork
agent: Explore
allowed-tools: Bash(gh *)
---

## Pull request context
- PR diff: !`gh pr diff`
- PR comments: !`gh pr view --comments`
- Changed files: !`gh pr diff --name-only`

## Your task
Summarize this pull request...
```

When the skill runs:

1. Each `` !`<command>` `` executes immediately (before Claude sees anything).
2. The output replaces the placeholder in the skill content.
3. Claude receives the fully-rendered prompt with actual data.

> "This is preprocessing, not something Claude executes. Claude only sees the final result."

This is what grounds a skill in your live working tree rather than what Claude can guess — for example, a `summarize-changes` skill built around `` !`git diff HEAD` ``.

## Single-pass semantics

> "Substitution runs once over the original file. Command output is inserted as plain text and is not re-scanned for further `` !`<command>` `` placeholders, so a command cannot emit a placeholder for a later pass to expand."

## Where the inline form is recognized

The inline form is recognized only when `!` appears at the **start of a line or immediately after whitespace**. If `!` follows another character, as in `` KEY=!`cmd` ``, the placeholder is left as literal text and the command does not run.

## Multi-line commands

For multiple commands, use a fenced block opened with `` ```! `` instead of the inline form:

````markdown
## Environment
```!
node --version
npm --version
git status --short
```
````

The shell used for both forms is controlled by the [`shell` frontmatter field](./skill-frontmatter.md) (`bash` default, or `powershell`). Command paths commonly use [`${CLAUDE_SKILL_DIR}`](./skill-arguments.md) so bundled scripts resolve from any working directory.

## Disabling shell execution by policy

To disable this behavior for skills and custom commands from user, project, plugin, or additional-directory sources, set `"disableSkillShellExecution": true` in [settings](./settings-files.md):

> "Each command is replaced with `[shell command execution disabled by policy]` instead of being run. Bundled and managed skills are not affected."

This is most useful in [managed settings](./settings-files.md), where users cannot override it. For a stronger, deterministic alternative to shell injection inside skills, see [Hooks](./hook-types.md).

## Related concepts

- [Skills](./skills.md) — the core concept
- [Skill Arguments and Substitutions](./skill-arguments.md) — the `$`-style substitutions that share this preprocessing step
- [Skill Frontmatter](./skill-frontmatter.md) — the `shell` and `allowed-tools` fields
- [Settings Files](./settings-files.md) — the `disableSkillShellExecution` policy setting
