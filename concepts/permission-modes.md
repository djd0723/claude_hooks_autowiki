---
type: concept
title: "Permission Modes"
created: 2026-06-29
updated: 2026-06-29
tags: [permissions, modes, security, bypass, sandbox]
source_count: 1
sources:
  - sources/clean/code-claude-com-docs-en-permissions-md.md
---

# Permission Modes

A permission mode is a session-wide preset that controls **how Claude Code approves tool calls** before any individual [permission rule](./permission-evaluation.md) is even consulted. Set the starting mode with `defaultMode` in your [settings files](./permission-settings.md); the `--permission-mode` CLI flag overrides it for one session.

## The modes

| Mode | Behavior |
| :--- | :------- |
| `default` | Standard behavior: prompts for permission on first use of each tool |
| `acceptEdits` | Automatically accepts file edits and common filesystem commands such as `mkdir`, `touch`, `mv`, and `cp` for paths in the working directory or `additionalDirectories` |
| `plan` | Plan Mode: Claude reads files and runs read-only shell commands to explore but doesn't edit your source files |
| `auto` | Auto-approves tool calls with background safety checks that verify actions align with your request. Currently a research preview |
| `dontAsk` | Auto-denies tools unless pre-approved via `/permissions` or `permissions.allow` rules |
| `bypassPermissions` | Skips permission prompts, except those forced by explicit `ask` rules |

## Circuit breakers in bypass mode

`bypassPermissions` is the most permissive mode, but it is not unconditional. Two safeguards always remain:

- **Explicit `ask` rules still force a prompt.** A mode cannot silence a rule you deliberately wrote to gate something.
- **Root- and home-directory removals still prompt.** Per the source: removals "targeting the filesystem root or home directory, such as `rm -rf /` and `rm -rf ~`, still prompt as a circuit breaker against model error."

The source also warns that bypass mode "skips permission prompts, including for writes to `.git`, `.config/git`, `.claude`, `.vscode`, `.idea`, `.husky`, `.cargo`, `.devcontainer`, `.yarn`, and `.mvn`. ... Only use this mode in isolated environments like containers or VMs where Claude Code can't cause damage."

## Locking modes out

To prevent the two permissive modes from being selected at all, set either flag to `"disable"` in any settings file:

- `permissions.disableBypassPermissionsMode` — blocks `bypassPermissions` (and the `--dangerously-skip-permissions` flag).
- `permissions.disableAutoMode` — blocks `auto`.

These "are most useful in [managed settings](./managed-settings.md) where they can't be overridden." Note that `disableBypassPermissionsMode` "works from any scope" — a user can even set it in their own settings to lock themselves out of bypass mode.

## How modes interact with sandboxing

When [sandboxing](./sandbox-settings.md) is enabled with `autoAllowBashIfSandboxed: true` (the default), sandboxed Bash commands run without prompting even under a bare `Bash` ask rule — the sandbox boundary substitutes for that whole-tool prompt. Content-scoped ask rules (e.g. `Bash(git push *)`), explicit deny rules, and the `rm`/`rmdir` circuit breakers still apply. See [Bash permission matching](./bash-permission-matching.md).

## Related concepts

- [Permission Evaluation](./permission-evaluation.md) — the deny→ask→allow algorithm that runs within a mode
- [Permission Settings](./permission-settings.md) — where `defaultMode` and the disable flags live
- [Managed Settings](./managed-settings.md) — locking the permissive modes org-wide
- [Sandbox Settings](./sandbox-settings.md) — OS-level Bash isolation that complements modes
