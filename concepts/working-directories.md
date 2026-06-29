---
type: concept
title: "Working Directories"
created: 2026-06-29
updated: 2026-06-29
tags: [permissions, working-directory, add-dir, cd, configuration, additionalDirectories]
source_count: 1
sources:
  - sources/clean/code-claude-com-docs-en-permissions-md.md
---

# Working Directories

By default Claude has file access only to the directory where you launched it. You can **extend** that access or **relocate** the session — and the two operations differ in an important way: extending grants file access but does **not** make the new directory a configuration root.

## Extending access

Three ways to add a directory to the set Claude can read and edit:

- **During startup**: the `--add-dir <path>` CLI argument
- **During session**: the `/add-dir` command
- **Persistent**: `additionalDirectories` in [settings files](./permission-settings.md)

Files in additional directories follow the same [permission rules](./permission-evaluation.md) as the original working directory — readable without prompts, edits following the current [permission mode](./permission-modes.md).

## Additional directories grant file access, not configuration

This is the trap. Adding a directory extends where Claude can read and edit; it does **not** turn that directory into a full configuration root. Most `.claude/` configuration is not discovered from additional directories — only a few types load as exceptions, and **only** for directories added via `--add-dir` / `/add-dir`. Directories listed in `permissions.additionalDirectories` grant **file access only** and load none of the configuration below.

Loaded from `--add-dir` directories:

| Configuration | Loaded from `--add-dir` |
| :------------ | :---------------------- |
| [Skills](./skills.md) in `.claude/skills/` | Yes, with live reload |
| [Subagents](./subagents.md) in `.claude/agents/` | Yes |
| Settings in `.claude/settings.json` / `.local.json` | `enabledPlugins` and `extraKnownMarketplaces` keys only |
| CLAUDE.md, `.claude/rules/`, `CLAUDE.local.md` | Only when `CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD=1` is set (`CLAUDE.local.md` also needs the `local` setting source, on by default) |

Commands and output styles are discovered from the cwd and its parents, `~/.claude/`, and managed settings. Hooks and other `settings.json` keys load from the cwd's `.claude/` folder with **no parent-directory fallback**, plus user and managed settings. To share configuration across projects, the source recommends user-level config in `~/.claude/`, packaging it as a [plugin](./plugins-vs-standalone.md), or launching Claude Code from the directory that holds the `.claude/` you want.

## Relocating the session with `/cd`

To change the session's **primary** working directory instead of adding another, use `/cd` (requires Claude Code v2.1.169+). Unlike `--add-dir`, it relocates the session: "the new directory's `CLAUDE.md` is loaded and `--resume` finds the session from there." Which targets `/cd` accepts is governed by [`Cd` permission rules](./file-permission-patterns.md) — and because `/cd` is user-run, not model-invocable, those rules apply only to your own `/cd` calls.

## Related concepts

- [Permission Settings](./permission-settings.md) — where `additionalDirectories` lives
- [Permission Modes](./permission-modes.md) — `acceptEdits` auto-accepts edits in the working and additional directories
- [File Permission Patterns](./file-permission-patterns.md) — `Cd` rules and the project-root anchor for paths
- [Configuration Scopes](./configuration-scopes.md) — why `.claude/` configuration is tied to specific roots
- [Configuration loading from settings files](./settings-files.md) — the discovery rules for hooks and settings keys
- [Large-Codebase Playbook](./large-codebase-playbook.md) — initializing in the right directory as a scale strategy
