---
type: summary
title: "Hooks reference — event schemas, JSON I/O, and decision control"
slug: code-claude-com-docs-en-hooks-md
created: 2026-06-29
updated: 2026-06-29
tags: [hooks, reference, json-schema, lifecycle, configuration, decision-control, async]
sources:
  - sources/clean/code-claude-com-docs-en-hooks-md.md
---

# Summary: Hooks reference

The authoritative reference for Claude Code hook schemas, configuration options, JSON input/output formats, and advanced features. Companion to the [hooks guide summary](hooks-guide.md), which covers setup recipes and troubleshooting.

## Key thesis from source

> "Use this reference to look up event schemas, configuration options, JSON input/output formats, and advanced features like async hooks, HTTP hooks, and MCP tool hooks. If you're setting up hooks for the first time, start with the guide instead."

## What this source covers

- Full table of all 30+ [hook lifecycle events](../concepts/hook-lifecycle-events.md) with when/cadence
- Configuration schema: locations, [matcher patterns](../concepts/hook-matchers.md), and handler fields
- Exec form vs shell form for command hooks; `asyncRewake` background hooks
- Complete [JSON input/output protocol](../concepts/hook-input-output.md): common fields, exit codes, JSON output fields
- Per-event blocking table and [decision control](../concepts/hook-decision-control.md) patterns
- Individual event sections with annotated JSON examples

## Hook cadences

Events fall into three cadences:

| Cadence | Events |
| :------ | :----- |
| Once per session | `SessionStart`, `SessionEnd`, `Setup` |
| Once per turn | `UserPromptSubmit`, `UserPromptExpansion`, `Stop`, `StopFailure` |
| Every tool call in the agentic loop | `PreToolUse`, `PermissionRequest`, `PermissionDenied`, `PostToolUse`, `PostToolUseFailure`, `PostToolBatch` |
| Async / standalone | `Notification`, `MessageDisplay`, `SubagentStart`, `SubagentStop`, `TaskCreated`, `TaskCompleted`, `TeammateIdle`, `InstructionsLoaded`, `ConfigChange`, `CwdChanged`, `FileChanged`, `WorktreeCreate`, `WorktreeRemove`, `PreCompact`, `PostCompact`, `Elicitation`, `ElicitationResult` |

## Matcher rules per event

Matchers evaluate differently per event. A string of only letters/digits/`_`/spaces/`,`/`|` is an exact or OR-list match; anything else is a JavaScript regex.

Key per-event matcher targets:

| Event(s) | Matches on |
| :------- | :--------- |
| `PreToolUse`, `PostToolUse`, `PermissionRequest`, `PermissionDenied` | tool name |
| `SessionStart` | `startup`, `resume`, `clear`, `compact` |
| `SessionEnd` | `clear`, `resume`, `logout`, `prompt_input_exit`, `bypass_permissions_disabled`, `other` |
| `Notification` | notification type (e.g., `permission_prompt`, `idle_prompt`) |
| `SubagentStart`, `SubagentStop` | agent type name |
| `FileChanged` | literal filenames (watch list) |
| `StopFailure` | error type |
| `InstructionsLoaded` | load reason (`session_start`, `nested_traversal`, `path_glob_match`, `include`, `compact`) |

Events with no matcher support: `UserPromptSubmit`, `PostToolBatch`, `Stop`, `TeammateIdle`, `TaskCreated`, `TaskCompleted`, `WorktreeCreate`, `WorktreeRemove`, `CwdChanged`.

## Exec form vs shell form

> "A command hook runs as exec form when `args` is set, and shell form when `args` is omitted."

- **Exec form** (`args` present): Claude Code spawns the executable directly — no shell, no tokenization, path placeholders substituted verbatim. Required for hooks referencing `${CLAUDE_PLUGIN_ROOT}` or other path placeholders with spaces.
- **Shell form** (`args` absent): command string passed to `sh -c` (macOS/Linux) or Git Bash / PowerShell (Windows). Supports pipes, `&&`, redirects, and glob expansion.

On Windows, `.cmd`/`.bat` shims (npm, eslint, etc.) cannot be spawned in exec form — invoke the underlying script with `node` directly instead.

## Async hooks

Command hooks support two background flags:

| Field | Behavior |
| :---- | :------- |
| `async: true` | Runs in the background; does not block the session |
| `asyncRewake: true` | Runs in background; wakes Claude on exit code 2, showing stderr as a system reminder so Claude can react |

> "If `true`, runs in the background and wakes Claude on exit code 2. Implies `async`. The hook's stderr, or stdout if stderr is empty, is shown to Claude as a system reminder so it can react to a long-running background failure."

## Common JSON input fields

Every hook event receives these on stdin (or as HTTP POST body):

| Field | Description |
| :---- | :---------- |
| `session_id` | Session identifier |
| `transcript_path` | Path to conversation JSONL |
| `cwd` | Working directory when hook fires |
| `permission_mode` | Active permission mode (on supported events) |
| `effort` | Object with `level` field (`low`/`medium`/`high`/`xhigh`/`max`) |
| `hook_event_name` | Name of the firing event |
| `agent_id` / `agent_type` | Present when hook fires inside a subagent |

## JSON output fields (universal)

| Field | Default | Description |
| :---- | :------ | :---------- |
| `continue` | `true` | If `false`, Claude stops entirely. Takes precedence over all event-specific decisions |
| `stopReason` | none | Message to user when `continue` is `false`; not shown to Claude |
| `suppressOutput` | `false` | Hides hook stdout from transcript (still in debug log) |
| `systemMessage` | none | Warning message shown to the user |
| `terminalSequence` | none | OSC escape sequence (OSC 0/1/2/9/99/777, BEL) emitted via Claude Code's own terminal path. Hooks cannot write to `/dev/tty` directly |

The `additionalContext` field is event-specific, returned inside `hookSpecificOutput.additionalContext`.

## Decision control summary

| Events | Decision pattern |
| :----- | :--------------- |
| `UserPromptSubmit`, `UserPromptExpansion`, `PostToolUse`, `PostToolUseFailure`, `PostToolBatch`, `Stop`, `SubagentStop`, `ConfigChange`, `PreCompact` | Top-level `{"decision": "block", "reason": "..."}` |
| `PreToolUse` | `hookSpecificOutput.permissionDecision`: `allow`/`deny`/`ask`/`defer`; also supports `updatedInput` and `additionalContext` |
| `PermissionRequest` | `hookSpecificOutput.decision.behavior`: `allow`/`deny`; also supports `updatedInput` |
| `PermissionDenied` | `hookSpecificOutput.retry: true` tells Claude it may retry |
| `WorktreeCreate` | Hook prints path on stdout; missing/error path fails creation |
| `Elicitation`, `ElicitationResult` | `hookSpecificOutput.action`: `accept`/`decline`/`cancel`; `content` for form values |
| `MessageDisplay` | `hookSpecificOutput.displayContent` replaces rendered text (transcript unchanged) |
| `SessionStart`, `Setup`, `SubagentStart` | Context-only: `additionalContext`, no blocking |
| `TeammateIdle`, `TaskCreated`, `TaskCompleted` | Exit code 2 blocks; or `{"continue": false, "stopReason": "..."}` |

When multiple `PreToolUse` hooks return conflicting decisions, precedence is: `deny > defer > ask > allow`.

## MessageDisplay event

Runs while assistant message text streams to screen. Key constraints:

> "MessageDisplay is display-only: the replacement text changes only what is rendered on screen. The transcript and what Claude sees keep the original text, so Claude never sees the replacement."

- Default timeout: 10 seconds
- In Agent SDK / `claude -p` runs: fires once per message (after completion) with full text in `delta`
- `displayContent` in `hookSpecificOutput` replaces the rendered batch; omit to show original

Use for: stripping markdown, redacting API keys/hostnames, transforming display for embedded applications.

## PreToolUse tool input schemas

The `tool_input` field varies by tool. Key schemas documented in the reference:

| Tool | Key input fields |
| :--- | :--------------- |
| `Bash` | `command`, `description`, `timeout`, `run_in_background` |
| `Write` | `file_path`, `content` |
| `Edit` | `file_path`, `old_string`, `new_string`, `replace_all` |
| `Read` | `file_path`, `offset`, `limit` |
| `Glob` | `pattern`, `path` |
| `Grep` | `pattern`, `path`, `glob`, `output_mode`, `-i`, `multiline` |
| `WebFetch` | `url`, `prompt` |
| `WebSearch` | `query`, `allowed_domains`, `blocked_domains` |
| `Agent` | `prompt`, `description`, `subagent_type`, `model` |
| `AskUserQuestion` | `questions` array, optional `answers` object |
| `ExitPlanMode` | `plan`, `planFilePath`, `allowedPrompts` |

For `PostToolUse` on an `Agent` call, `tool_response` contains usage telemetry: `totalTokens`, `totalDurationMs`, `totalToolUseCount`, `usage`, `resolvedModel`.

## defer mechanism (PreToolUse)

`permissionDecision: "defer"` pauses Claude at a tool call for Agent SDK / `claude -p` integrations:

> "It lets that calling process pause Claude at a tool call, collect input through its own interface, and resume where it left off."

Exit output includes `stop_reason: "tool_deferred"` and a `deferred_tool_use` payload. Resume with `claude -p --resume <session-id>`; the hook then returns `"allow"` with `updatedInput`. Only works when Claude makes a single tool call per turn.

## SessionStart special outputs

In addition to `additionalContext`, `SessionStart` supports:

| Field | Description |
| :---- | :---------- |
| `initialUserMessage` | First user message in non-interactive (`-p`) mode |
| `sessionTitle` | Sets session title (equivalent to `/rename`) |
| `watchPaths` | Absolute paths to watch for `FileChanged` events this session |
| `reloadSkills` | `true` to re-scan skills directories after SessionStart hooks finish |

`CLAUDE_ENV_FILE` is available to `SessionStart`, `Setup`, `CwdChanged`, and `FileChanged` hooks for persisting environment variables to subsequent Bash commands.

## Enterprise / managed hooks

> "Enterprise administrators can use `allowManagedHooksOnly` to block user, project, and plugin hooks. Hooks from plugins force-enabled in managed settings `enabledPlugins` are exempt."

`disableAllHooks: true` in user/project/local settings cannot disable managed-policy hooks; only `disableAllHooks` at the managed level can.

## Relationship to concept pages

This reference is the primary source for:
- [Hook lifecycle events](../concepts/hook-lifecycle-events.md) — full event table with cadences
- [Hook matchers](../concepts/hook-matchers.md) — per-event matcher rules and MCP tool matching
- [Hook input/output](../concepts/hook-input-output.md) — JSON schemas, exit codes, JSON output fields
- [Hook exit codes](../concepts/hook-exit-codes.md) — per-event blocking table
- [Hook decision control](../concepts/hook-decision-control.md) — decision patterns per event type
- [Hook types](../concepts/hook-types.md) — command/HTTP/MCP tool/prompt/agent handler fields
- [Hook scope](../concepts/hook-scope.md) — location table and enterprise scope
