---
type: concept
title: "File Permission Patterns"
created: 2026-06-29
updated: 2026-06-29
tags: [permissions, read, edit, gitignore, symlinks, paths, cd]
source_count: 1
sources:
  - sources/clean/code-claude-com-docs-en-permissions-md.md
---

# File Permission Patterns

`Read` and `Edit` [permission rules](./permission-evaluation.md) gate file access by **path pattern**, using [gitignore](https://git-scm.com/docs/gitignore) semantics. This page covers how those patterns anchor, how symlinks are checked, and the related `Cd` rules. For which tools each rule format governs, see [Tool Permission Rules](./tool-permission-rules.md).

## Which tools the rules cover

- `Edit` rules "apply to all built-in tools that edit files."
- `Read` rules apply on a best-effort basis to all built-in file-reading tools (Grep, Glob), to `@file` mentions in prompts, and to the selection/open-file context a connected IDE shares.

Deny rules for Read and Edit also cover the **file commands Claude Code recognizes in Bash** — `cat`, `head`, `tail`, `sed`. They do **not** cover arbitrary subprocesses that open files indirectly (a Python or Node script). For OS-level enforcement across all processes, [enable the sandbox](./sandbox-settings.md).

## The four anchor types

A pattern's leading characters decide what it is relative to:

| Pattern | Meaning | Example |
| :------ | :------ | :------ |
| `//path` | Absolute path from filesystem root | `Read(//Users/alice/secrets/**)` |
| `~/path` | Path from home directory | `Read(~/Documents/*.pdf)` |
| `/path` | Path relative to **project root** | `Edit(/src/**/*.ts)` |
| `path` or `./path` | Path relative to current directory | `Read(*.env)` |

> A pattern like `/Users/alice/file` isn't an absolute path. It's relative to the project root. Use `//Users/alice/file` for absolute paths.

On Windows, paths are normalized to POSIX before matching (`C:\Users\alice` → `/c/Users/alice`); use `//c/**/.env` for a drive, `//**/.env` across all drives.

## Anchor determines reach

A rule only matches files under its anchor, so the anchor decides how far a deny rule reaches. Bare filenames follow gitignore semantics and match at **any depth** — `Read(.env)` and `Read(**/.env)` are equivalent and block any `.env` at or under the current directory (but not one in a parent directory or another project). To block any `.env` anywhere on the filesystem, anchor at root: `Read(//**/.env)`.

In gitignore patterns, `*` matches within a single path segment and `**` matches across directories. To allow all file access, use the bare tool name with no parentheses: `Read`, `Edit`, or `Write`.

## Symlinks are checked twice

When Claude accesses a symlink, rules check **both** the symlink path and the file it resolves to — and allow vs. deny treat the pair differently:

- **Allow rules** apply only when **both** the symlink path and its target match. A symlink inside an allowed directory pointing outside it still prompts.
- **Deny rules** apply when **either** the symlink path or its target matches.

Example: with `Read(./project/**)` allowed and `Read(~/.ssh/**)` denied, a symlink `./project/key` → `~/.ssh/id_rsa` is blocked — the target fails the allow rule and matches the deny rule.

## Cd rules

`Cd` rules control which directories the `/cd` command can move the session to. `Cd` is **not a model-invocable tool** — Claude can't call it; the rules apply only when you run `/cd` yourself (see [working directories](./working-directories.md)).

- A bare `Cd` deny disables `/cd` entirely; `Cd(<path-pattern>)` deny blocks matching targets. Deny rules check every spelling of the target, including each symlink hop.
- Adding **any** `Cd` allow rule switches `/cd` to **allowlist mode** — the resolved target must match an allow rule or `/cd` refuses. With no `Cd` rules, `/cd` prompts to trust an unfamiliar directory.
- `Cd` patterns share the `//`, `~/`, `/` anchors but match the **whole directory path** (not gitignore-style): `*` matches one segment, `**` matches across segments, and a trailing `/**` also matches its named root.

## Related concepts

- [Permission Evaluation](./permission-evaluation.md) — the deny→ask→allow order these matches feed into
- [Tool Permission Rules](./tool-permission-rules.md) — that `Read(...)` governs Grep/Glob and `Edit(...)` governs Write/NotebookEdit
- [Working Directories](./working-directories.md) — `--add-dir`, `/add-dir`, and `/cd` that these rules interact with
- [Sandbox Settings](./sandbox-settings.md) — OS-level path enforcement that merges with Read/Edit deny rules
- [File Tool Behavior](./file-tool-behavior.md) — how the Read/Edit/Write tools behave once permitted
