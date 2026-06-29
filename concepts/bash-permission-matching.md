---
type: concept
title: "Bash Permission Matching"
created: 2026-06-29
updated: 2026-06-29
tags: [permissions, bash, powershell, wildcards, compound-commands, wrappers]
source_count: 1
sources:
  - sources/clean/code-claude-com-docs-en-permissions-md.md
---

# Bash Permission Matching

Bash is the hardest tool to gate safely, because a shell command can wrap, chain, or rewrite itself. Claude Code's Bash [permission rules](./permission-evaluation.md) therefore have matching mechanics no other tool needs. PowerShell uses the same shape. For what the Bash tool actually does once a call is approved, see [Bash execution model](./bash-execution-model.md).

## Wildcards and the word boundary

Bash rules support glob `*` at any position — beginning, middle, or end. A single `*` matches any sequence including spaces, so one wildcard can span multiple arguments (`Bash(git *)` matches `git log --oneline --all`).

The decisive subtlety is the **space before a trailing `*`**:

> `Bash(ls *)` matches `ls -la` but not `lsof`, while `Bash(ls*)` matches both.

A space before `*` enforces a word boundary (the prefix must be followed by a space or end-of-string). The `:*` suffix is an equivalent way to write a trailing wildcard — `Bash(ls:*)` equals `Bash(ls *)` — and the permission dialog writes the space-separated form when you select "Yes, don't ask again." The `:*` form is recognized **only at the end**; mid-pattern colons are literal.

## Compound commands

Claude Code is aware of shell operators, so a rule like `Bash(safe-cmd *)` does **not** authorize `safe-cmd && other-cmd`. The recognized separators are `&&`, `||`, `;`, `|`, `|&`, `&`, and newlines; **a rule must match each subcommand independently.**

When you approve a compound command with "Yes, don't ask again," Claude Code saves a **separate rule per subcommand** rather than one rule for the full string — approving `git status && npm test` saves a rule for `npm test`. Up to 5 rules may be saved for a single compound command.

## Process wrappers

Before matching, Claude Code **strips a fixed set of process wrappers** so a rule covers wrapped invocations:

- Stripped wrappers: `timeout`, `time`, `nice`, `nohup`, `stdbuf` — so `Bash(npm test *)` also matches `timeout 30 npm test`.
- **Bare `xargs`** is stripped (`Bash(grep *)` matches `xargs grep pattern`) — **but only with no flags**. `xargs -n1 grep pattern` is matched as an `xargs` command, so inner-command rules don't cover it.

The list is built-in and **not configurable**. Critically, environment runners — `direnv exec`, `devbox run`, `mise exec`, `npx`, `docker exec` — are **not** stripped. Because they execute their arguments, `Bash(devbox run *)` matches whatever follows `run`, including `devbox run rm -rf .`. Approve a specific runner+inner pair instead: `Bash(devbox run npm test)`.

Exec wrappers `watch`, `setsid`, `ionice`, `flock` **always prompt** and can't be prefix-approved. The same applies to `find` with `-exec` or `-delete`. Write an exact-match rule for the full command string to approve these.

## Built-in read-only commands

A built-in set of commands runs **without a prompt in every mode**: `ls`, `cat`, `echo`, `pwd`, `head`, `tail`, `grep`, `find`, `wc`, `which`, `diff`, `stat`, `du`, `cd`, and read-only forms of `git`. The set is not configurable; add an `ask`/`deny` rule to require a prompt for one.

- Unquoted globs are allowed when **every flag is read-only** (`ls *.ts`, `wc -l src/*.py` run freely).
- Write- or exec-capable commands (`find`, `sort`, `sed`, `git`) still prompt with an unquoted glob, because it could expand to a flag like `-delete`.
- A `cd` into your [working directory](./working-directories.md) or an additional directory is read-only; `cd packages/api && ls` runs without a prompt. **Combining `cd` with `git`** in one compound command always prompts.

## Why argument-constraining patterns are fragile

The source warns that patterns trying to constrain arguments — e.g. `Bash(curl http://github.com/ *)` — break on options-before-URL, different protocol, redirects, variables, and extra spaces. For reliable URL filtering it recommends: deny `curl`/`wget` and use `WebFetch(domain:...)`; or a PreToolUse hook that validates URLs; or `CLAUDE.md` guidance paired with one of those. Note: "using WebFetch alone doesn't prevent network access. If Bash is allowed, Claude can still use `curl`, `wget`, or other tools to reach any URL."

## PowerShell

PowerShell rules use the same shape (positional `*`, `:*` suffix, bare `PowerShell`/`PowerShell(*)` matches all). Differences:

- **Aliases are canonicalized before matching** — `PowerShell(Get-ChildItem *)` also matches `gci`, `ls`, and `dir`. Matching is case-insensitive.
- Claude Code parses the PowerShell AST; pipeline `|`, separator `;`, and (on PowerShell 7+) chain operators `&&`/`||` split compounds — every subcommand must match.

## Related concepts

- [Permission Evaluation](./permission-evaluation.md) — the deny→ask→allow order these matches feed into
- [Bash Execution Model](./bash-execution-model.md) — what the Bash tool does once a call is approved
- [File Permission Patterns](./file-permission-patterns.md) — Read/Edit also gate the file commands Bash recognizes
- [Permission Modes](./permission-modes.md) — `acceptEdits` and sandbox interactions with Bash
