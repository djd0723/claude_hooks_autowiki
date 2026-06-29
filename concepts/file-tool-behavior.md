---
type: concept
title: "File Tool Behavior"
created: 2026-06-29
updated: 2026-06-29
tags: [tools, read, edit, write, glob, grep, notebook, gitignore]
source_count: 1
sources:
  - sources/clean/code-claude-com-docs-en-tools-reference-md.md
---

# File Tool Behavior

The file-facing [built-in tools](./built-in-tools.md) — `Read`, `Edit`, `Write`, `NotebookEdit`, `Glob`, and `Grep` — each have behavior worth knowing because it shapes how reliably an edit lands and what a search will (and won't) find.

## Read

`Read` takes a file path and returns the contents **with line numbers**; Claude is instructed to always pass absolute paths.

- A whole-file read that exceeds the token limit returns the first page with a `PARTIAL view` notice telling Claude how to continue with `offset` and `limit`. A read that *already* passes an explicit `offset`/`limit` and still overflows returns an error.
- **Images** (PNG, JPG, …) come back as visual content Claude can see, not raw bytes; large images are downscaled to fit the model's size limits.
- **PDFs** under 10 pages read whole; longer ones read in ranges via a `pages` parameter (e.g. `"1-5"`), up to 20 pages at a time.
- **Jupyter notebooks** (`.ipynb`) return all cells with their outputs.
- `Read` reads files only, not directories — Claude uses `ls` via [Bash](./bash-execution-model.md) to list a directory.

## The read-before-edit constraint

`Edit` and `Write` both require that Claude has already read an existing file in the current conversation before changing it. This is an **edit-eligibility** rule, separate from permissions.

`Edit` performs **exact string replacement** (no regex, no fuzzy matching). Three checks must pass:

1. **Read-before-edit** — Claude must have read the file this conversation, and it must not have changed on disk since. This runs first, before any matching.
2. **Match** — `old_string` must appear exactly as written; a single whitespace difference misses.
3. **Uniqueness** — `old_string` must appear exactly once, or Claude supplies more surrounding context, or sets `replace_all: true`.

`Write` creates or overwrites with full content (no append/merge). Overwriting an **existing** file requires a prior read in the conversation; writing a brand-new file does not. For partial changes, Claude uses `Edit` rather than `Write`.

**Viewing a file with Bash can satisfy read-before-edit** — but only for `cat`, `head`, `tail`, `sed -n 'X,Yp'`, `grep`, `egrep`, or `fgrep` on a single file with **no pipes or redirects**. Piped output and other commands don't count.

> A subtle asymmetry: the Bash commands that satisfy read-before-edit are **not** the same set checked against [Read/Edit deny rules](./tool-permission-rules.md). For example, `egrep` and `fgrep` count for read-before-edit but are *not* checked against Read deny rules; and deny rules don't cover a Python or Node script that opens files itself. For OS-level enforcement across every process, enable the sandbox.

## NotebookEdit

`NotebookEdit` modifies a Jupyter notebook **one cell at a time**, targeting cells by `cell_id` — it does not do cross-notebook string replacement the way `Edit` does on plain files. Three modes:

- `replace` — overwrite the cell's source (default).
- `insert` — add a new cell after the target (with no `cell_id`, it goes at the start); requires `cell_type` of `code` or `markdown`.
- `delete` — remove the target cell.

Permission rules use the `Edit(...)` path format — `Edit(notebooks/**)` covers `NotebookEdit` calls there.

## Glob vs Grep

Both find things in the tree, but along different axes — and they treat `.gitignore` **oppositely**.

| | `Glob` | `Grep` |
| :-- | :----- | :----- |
| Finds | files by **name** pattern (`**/*.ts`, `*.{json,yaml}`) | **lines** inside files (ripgrep regex) |
| Sort / cap | by modification time, capped at 100 files (truncation flag if hit) | output modes: `files_with_matches` (default), `content`, `count` |
| `.gitignore` | **ignored by default** — finds gitignored files too | **respected** — gitignored files skipped |
| Override | `CLAUDE_CODE_GLOB_NO_IGNORE=false` makes Glob respect `.gitignore` | pass a gitignored file's path directly to search it |

`Grep` uses **ripgrep's regex syntax**, not POSIX grep, so metacharacters need escaping — finding `interface{}` in Go takes the pattern `interface\{\}`. Results can be scoped with `glob` (e.g. `**/*.tsx`) or `type` (e.g. `py`, `rust`), and `multiline: true` matches across line boundaries.

## Related concepts

- [Built-in Tools](./built-in-tools.md) — the full tool catalog
- [Tool Permission Rules](./tool-permission-rules.md) — how `Read(...)` and `Edit(...)` rules map onto these tools
- [Bash Execution Model](./bash-execution-model.md) — the Bash behavior that interacts with read-before-edit
