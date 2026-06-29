---
type: summary
title: "Hooks Guide — Automate actions with hooks"
slug: code-claude-com-docs-en-hooks-guide-md
created: 2026-06-29
updated: 2026-06-29
tags: [hooks, automation, lifecycle, configuration, recipes, troubleshooting]
sources:
  - sources/clean/code-claude-com-docs-en-hooks-guide-md.md
---

# Summary: Automate actions with hooks

Official Claude Code guide covering how to set up and use hooks — user-defined shell commands that run automatically at lifecycle points.

## Key thesis from source

> "Hooks are user-defined shell commands that execute at specific points in Claude Code's lifecycle. They provide deterministic control over Claude Code's behavior, ensuring certain actions always happen rather than relying on the LLM to choose to run them."

## What this source covers

- How hooks work: events, matchers, input/output/exit codes
- The 30+ [hook lifecycle events](../concepts/hook-lifecycle-events.md) and when they fire
- Five [hook types](../concepts/hook-types.md): command, http, mcp_tool, prompt, agent
- [Matcher patterns](../concepts/hook-matchers.md) and the `if` field for argument-level filtering
- [Exit codes and JSON structured output](../concepts/hook-exit-codes.md) for communicating decisions
- [Hook scope and location](../concepts/hook-scope.md): user, project, plugin, skill
- Seven ready-to-use recipes with copy-paste JSON configs
- Prompt-based and agent-based hooks for judgment-requiring decisions
- HTTP hooks for posting event data to external endpoints
- Limitations, permission mode interactions, and debug techniques

## Seven key recipes

Each recipe maps to specific event(s) and includes a complete config block:

| Recipe | Event(s) | What it does |
| :----- | :------- | :----------- |
| Notifications | `Notification` | Desktop alert when Claude waits for input |
| Auto-format | `PostToolUse` + `Edit\|Write` matcher | Run Prettier after every file edit |
| Block protected files | `PreToolUse` + `Edit\|Write` matcher | Exit 2 to block `.env`, `.git/`, etc. |
| Re-inject after compaction | `SessionStart` + `compact` matcher | Restore context lost during compaction |
| Audit config changes | `ConfigChange` | Append every settings change to a log |
| Reload environment | `SessionStart` + `CwdChanged` | Run `direnv export bash > "$CLAUDE_ENV_FILE"` |
| Auto-approve prompts | `PermissionRequest` + tool matcher | Return `{"behavior": "allow"}` JSON to skip dialog |

## Prompt-based hooks

Use `type: "prompt"` when a decision requires judgment:

> "Instead of running a shell command, Claude Code sends your prompt and the hook's input data to a Claude model (Haiku by default) to make the decision."

The model returns `{"ok": true}` or `{"ok": false, "reason": "..."}`. On `false`:
- `Stop` / `SubagentStop`: reason is fed back so Claude keeps working
- `PreToolUse`: tool call is denied, reason returned as tool error
- `PostToolUse`, `UserPromptSubmit`, etc.: turn ends and reason appears in chat

## Agent-based hooks

Use `type: "agent"` when verification requires inspecting files or running commands:

> "Unlike prompt hooks which make a single LLM call, agent hooks spawn a subagent that can read files, search code, and use other tools to verify conditions before returning a decision."

Default timeout: 60 seconds, up to 50 tool-use turns. Same `ok`/`reason` response format as prompt hooks.

## HTTP hooks

Use `type: "http"` to POST event data to an endpoint instead of running a shell command:

> "HTTP hooks are useful when you want a web server, cloud function, or external service to handle hook logic: for example, a shared audit service that logs tool use events across a team."

Header values support `$VAR_NAME` interpolation, but only for variables listed in `allowedEnvVars`. HTTP status codes alone cannot block actions — return JSON `hookSpecificOutput` in a 2xx body.

## The `if` field (v2.1.85+)

Beyond the `matcher` (which filters at group level by tool name), the `if` field filters individual hooks by tool name **and arguments**:

> "The `if` field uses permission rule syntax to filter hooks by tool name and arguments together, so the hook process only spawns when the tool call matches."

Example: `"if": "Bash(git *)"` runs the hook only when Claude runs a git command, not every Bash call. The filter fails open (runs hook anyway) when the Bash command can't be parsed — use the permission system for hard enforcements.

Only works on tool events: `PreToolUse`, `PostToolUse`, `PostToolUseFailure`, `PermissionRequest`, `PermissionDenied`.

## Permission mode interactions

Key invariant from source:

> "PreToolUse hooks fire before any permission-mode check. A hook that returns `permissionDecision: 'deny'` blocks the tool even in `bypassPermissions` mode."

The reverse does not hold: a hook returning `"allow"` cannot override deny rules from settings.

## Key limitations

- Hooks communicate via stdout/stderr/exit codes only — cannot trigger `/` commands or tool calls
- Timeouts: command/http/mcp_tool = 10 min (lowered to 30s for `UserPromptSubmit`, 10s for `MessageDisplay`); prompt = 30s; agent = 60s
- `PostToolUse` hooks cannot undo actions (tool already executed)
- `PermissionRequest` hooks do not fire in non-interactive (`-p`) mode — use `PreToolUse` instead
- `Stop` hooks hit a block cap of 8 consecutive blocks; check `stop_hook_active` field to avoid this

## Debugging

- `/hooks` menu shows all configured hooks grouped by event (read-only)
- Transcript view (`Ctrl+O`) shows one-line summary per hook that fired
- Full debug log: `claude --debug-file /tmp/claude.log` then `tail -f /tmp/claude.log`
- Common pitfall: shell profile `echo` statements corrupt hook JSON output; guard with `if [[ $- == *i* ]]`

## Scope

This is the **tutorial guide**, not the full reference. For full event schemas, JSON formats, async hooks, and MCP tool hooks, the source points to the Hooks reference (`/en/hooks`).
