---
type: concept
title: "Bash Execution Model"
created: 2026-06-29
updated: 2026-06-29
tags: [tools, bash, powershell, shell, background, timeout, environment]
source_count: 1
sources:
  - sources/clean/code-claude-com-docs-en-tools-reference-md.md
---

# Bash Execution Model

The [`Bash` tool](./built-in-tools.md) runs each command in a **separate process**. What persists between commands, what the limits are, and how to run things in the background all follow from that.

## What persists between commands

- **Working directory carries over** — a `cd` in the main session affects later Bash commands, *as long as it stays inside the project directory or an [additional working directory](./permission-settings.md)* added with `--add-dir`, `/add-dir`, or `additionalDirectories`. **Subagent sessions never carry over** working-directory changes.
  - If `cd` lands outside those directories, Claude Code resets to the project directory and appends `Shell cwd was reset to <dir>` to the result.
  - Set `CLAUDE_BASH_MAINTAIN_PROJECT_WORKING_DIR=1` to disable carry-over so every command starts in the project directory.
- **Environment variables do *not* persist** — an `export` in one command is gone by the next. To make env vars persist, set `CLAUDE_ENV_FILE` to a shell script before launching, or populate it from a [SessionStart hook](./hook-lifecycle-events.md).
- **Aliases and shell functions *do* persist** — at session start Claude Code sources `~/.zshrc`, `~/.bashrc`, or `~/.profile` (per your shell), captures the resulting aliases, functions, and options, and applies them to every Bash command.

Activate your virtualenv or conda environment *before* launching Claude Code.

## Limits

Two limits bound each command, both overridable via [environment variables](./permission-settings.md):

| Limit | Default | Ceiling | Override |
| :---- | :------ | :------ | :------- |
| Timeout | 2 minutes | 10 minutes (via the `timeout` parameter) | `BASH_DEFAULT_TIMEOUT_MS`, `BASH_MAX_TIMEOUT_MS` |
| Output length | 30,000 chars | 150,000 chars (hard) | `BASH_MAX_OUTPUT_LENGTH` |

When output exceeds the limit, Claude Code saves the full output to a file in the session directory and gives Claude the path plus a short preview from the start — Claude reads or searches that file for the rest.

## Background tasks

For long-running processes (dev servers, watch builds), Claude can set `run_in_background: true` to start the command as a background task and keep working while it runs. List and stop background tasks with `/tasks`.

## PowerShell

The `PowerShell` tool runs PowerShell commands natively instead of routing through Git Bash.

- **Availability:** auto-enabled on Windows without Git Bash; rolling out progressively on Windows *with* Git Bash; opt-in on Linux/macOS/WSL (requires PowerShell 7+ / `pwsh` on `PATH`).
- **Enable** with `CLAUDE_CODE_USE_POWERSHELL_TOOL=1` (in the environment or `settings.json`); set it to `0` on Windows to opt out.
- Claude Code spawns PowerShell with `-ExecutionPolicy Bypass` at **process scope only**, so `.ps1` scripts work without changing machine policy — but Group Policy `MachinePolicy`/`UserPolicy` lockdowns still apply. Set `CLAUDE_CODE_POWERSHELL_RESPECT_EXECUTION_POLICY=1` to respect the machine's effective policy.
- Three settings route specific surfaces through PowerShell: `"defaultShell": "powershell"` (interactive `!` commands), `"shell": "powershell"` on a [command hook](./hook-types.md) (works regardless of the tool flag, since hooks spawn PowerShell directly), and `shell: powershell` in [skill frontmatter](./skill-frontmatter.md).
- The same working-directory reset behavior (and `CLAUDE_BASH_MAINTAIN_PROJECT_WORKING_DIR`) applies to PowerShell.
- *Preview limitations:* PowerShell profiles are not loaded, and Windows sandboxing is not supported.

## Monitor

`Monitor` is the background-watch counterpart to Bash — it runs a command and feeds each output line back to Claude as it arrives, so Claude reacts mid-conversation. It uses the **same permission rules as Bash**, and is restricted to interactive CLI sessions (not available on Bedrock/Vertex/Foundry, or when telemetry is disabled). See [monitors vs command hooks](../comparisons/monitors-vs-command-hooks.md) for how it differs from a hook.

## Related concepts

- [Built-in Tools](./built-in-tools.md) — the full tool catalog
- [File Tool Behavior](./file-tool-behavior.md) — how `cat`/`head`/`grep` in Bash satisfy read-before-edit
- [Tool Permission Rules](./tool-permission-rules.md) — `Bash(...)` and `PowerShell(...)` command patterns
- [Monitors vs Command Hooks](../comparisons/monitors-vs-command-hooks.md) — the Monitor tool versus event-driven hooks
