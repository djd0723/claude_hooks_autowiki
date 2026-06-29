---
type: concept
title: "Skill Discovery"
created: 2026-06-29
updated: 2026-06-29
tags: [skills, discovery, scope, precedence, nested, live-reload]
source_count: 1
sources:
  - sources/clean/code-claude-com-docs-en-skills-md.md
---

# Skill Discovery

Where you store a [skill](./skills.md) determines who can use it.

| Location | Path | Applies to |
| :------- | :--- | :--------- |
| Enterprise | See [managed settings](./settings-files.md) | All users in your organization |
| Personal | `~/.claude/skills/<skill-name>/SKILL.md` | All your projects |
| Project | `.claude/skills/<skill-name>/SKILL.md` | This project only |
| Plugin | `<plugin>/skills/<skill-name>/SKILL.md` | Where the plugin is enabled |

## Precedence

When skills share a name across levels:

> "enterprise overrides personal, and personal overrides project. A skill at any of these levels also overrides a bundled skill with the same name."

For example, a `code-review` skill in your project's `.claude/skills/` replaces the bundled [`/code-review`](./bundled-skills.md). Plugin skills use a `plugin-name:skill-name` namespace, so they cannot conflict with other levels. If a skill and a `.claude/commands/` command share a name, the skill takes precedence.

## Discovery from parent and nested directories

Project skills load from `.claude/skills/` in your starting directory **and every parent directory up to the repository root**, so starting Claude in a subdirectory still picks up root-level skills.

Claude Code also discovers skills from nested `.claude/skills/` directories on demand: when Claude reads or edits a file in a subdirectory, that subdirectory's `.claude/skills/` becomes available. This lets a monorepo package provide its own skills that apply when working on that package.

When a nested skill shares a name with another skill, **both stay available**:

- The nested one appears under a directory-qualified name, e.g. `apps/web:deploy`.
- Its description says which directory it applies to.
- Claude picks the variant matching the files it is working on.

Typing `/deploy` runs the project-root skill; type the qualified name `/apps/web:deploy` to run the nested variant explicitly. This qualified-name behavior is mirrored in how [the command name is derived](./skill-frontmatter.md) for clashing nested skills.

## Skills from additional directories

The `--add-dir` flag and `/add-dir` command grant *file access* rather than configuration discovery — but skills are an **exception**:

> "`.claude/skills/` within an added directory is loaded automatically. This exception applies only to `--add-dir` and `/add-dir`."

The `permissions.additionalDirectories` setting in `settings.json` grants file access only and does **not** load skills. Other `.claude/` configuration (commands, output styles) is also not loaded from additional directories.

## Live change detection

Claude Code watches skill directories for file changes. Adding, editing, or removing a skill under `~/.claude/skills/`, the project `.claude/skills/`, or a `.claude/skills/` inside an `--add-dir` directory takes effect within the current session without restarting.

Two caveats:

- Creating a **top-level** skills directory that did not exist at session start requires restarting Claude Code so the new directory can be watched.
- Live change detection covers `SKILL.md` text only. For a skill folder that is also a [plugin](./plugin-installation-scopes.md), changes to `hooks/`, `.mcp.json`, `agents/`, and `output-styles/` need `/reload-plugins`.

## Related concepts

- [Skills](./skills.md) — what a skill is
- [Skill Frontmatter](./skill-frontmatter.md) — how location determines the command name
- [Skill Invocation Control](./skill-invocation-control.md) — `skillOverrides` and per-skill visibility
- [Plugin Installation Scopes](./plugin-installation-scopes.md) — skills-directory plugins and the `@skills-dir` suffix
- [Settings Files](./settings-files.md) — managed/enterprise settings that host org-wide skills
