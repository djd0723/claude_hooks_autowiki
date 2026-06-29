---
type: concept
title: "Hook Matchers"
created: 2026-06-29
updated: 2026-06-29
tags: [hooks, matchers, filtering, configuration]
source_count: 2
sources:
  - sources/clean/code-claude-com-docs-en-hooks-guide-md.md
  - sources/clean/code-claude-com-docs-en-hooks-md.md
---

# Hook Matchers

Matchers let you narrow down when a hook fires. Without a matcher, a hook fires on every occurrence of its event.

## The `matcher` field

The `matcher` is a string on the hook group that filters which occurrences of the event trigger the group. Each event type matches on a specific field:

| Event | What the matcher filters | Example values |
| :---- | :----------------------- | :------------- |
| `PreToolUse`, `PostToolUse`, `PostToolUseFailure`, `PermissionRequest`, `PermissionDenied` | tool name | `Bash`, `Edit\|Write`, `mcp__.*` |
| `SessionStart` | how the session started | `startup`, `resume`, `clear`, `compact` |
| `Setup` | which CLI flag triggered setup | `init`, `maintenance` |
| `SessionEnd` | why the session ended | `clear`, `resume`, `logout`, `prompt_input_exit`, `bypass_permissions_disabled`, `other` |
| `Notification` | notification type | `permission_prompt`, `idle_prompt`, `auth_success`, `elicitation_dialog`, `elicitation_complete`, `elicitation_response` |
| `SubagentStart` / `SubagentStop` | agent type | `general-purpose`, `Explore`, `Plan`, or custom agent names |
| `PreCompact` / `PostCompact` | what triggered compaction | `manual`, `auto` |
| `ConfigChange` | configuration source | `user_settings`, `project_settings`, `local_settings`, `policy_settings`, `skills` |
| `StopFailure` | error type | `rate_limit`, `overloaded`, `authentication_failed`, `oauth_org_not_allowed`, `billing_error`, `invalid_request`, `model_not_found`, `server_error`, `max_output_tokens`, `unknown` |
| `InstructionsLoaded` | load reason | `session_start`, `nested_traversal`, `path_glob_match`, `include`, `compact` |
| `Elicitation` / `ElicitationResult` | MCP server name | your configured MCP server names |
| `FileChanged` | literal filenames to watch | `.envrc\|.env` |
| `UserPromptExpansion` | command name | your skill or command names |
| `UserPromptSubmit`, `PostToolBatch`, `Stop`, `TeammateIdle`, `TaskCreated`, `TaskCompleted`, `WorktreeCreate`, `WorktreeRemove`, `CwdChanged`, `MessageDisplay` | no matcher support | always fires on every occurrence |

## Matcher syntax rules

How a matcher value is evaluated depends on its characters:

| Matcher value | Evaluated as |
| :------------ | :----------- |
| `"*"`, `""`, or omitted | Match all |
| Only letters, digits, `_`, spaces, `,`, and `\|` | Exact string, or list separated by `\|` or `,` (v2.1.191+) with optional surrounding whitespace |
| Contains any other character | JavaScript regular expression |

So `Bash` is an exact match; `Edit\|Write` matches either exactly; `^Notebook` is a regex matching any tool starting with Notebook.

> Note: `FileChanged` and `StopFailure` accept only `|` as a list separator and treat `,` as a literal character. All other events accept `|` or `,`.

**MCP tool naming convention**: `mcp__<server>__<tool>`, e.g. `mcp__github__search_repositories`. Use `mcp__github__.*` to match all tools from one server, or `mcp__.*__write.*` to match across servers. The `.*` is required — a matcher like `mcp__memory` contains only letters and underscores and is compared as an exact string, matching no tool.

## The `if` field — argument-level filtering

The `if` field uses permission rule syntax to filter by tool name *and* arguments together. It is evaluated per-hook (not per-group) and only spawns the hook process when the call matches. Requires Claude Code v2.1.85+.

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "if": "Bash(git *)",
            "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/check-git-policy.sh"
          }
        ]
      }
    ]
  }
}
```

How `Bash(git *)` evaluates across compound commands:

| `if` pattern | Bash command | Hook runs? | Why |
| :----------- | :----------- | :--------- | :-- |
| `Bash(git *)` | `git push` | yes | command name matches |
| `Bash(git *)` | `npm test && git push` | yes | each subcommand is checked; `git push` matches |
| `Bash(git *)` | `echo $(git log)` | yes | commands inside `$()` and backticks are checked |
| `Bash(git *)` | `echo $(date)` | no | no subcommand matches `git *` |
| `Bash(git push *)` | `echo $(date)` | yes | patterns specifying more than the command name run on `$()`, backticks, or `$VAR` |

The filter **fails open** — it runs the hook when the Bash command cannot be parsed. Use the permission system, not `if`, to enforce a hard allow/deny.

`if` only works on tool events: `PreToolUse`, `PostToolUse`, `PostToolUseFailure`, `PermissionRequest`, `PermissionDenied`. Adding it to any other event prevents the hook from running.

## Multiple hooks per event

When multiple hooks match the same event, every hook's command runs to completion before Claude Code merges the results. One hook returning `deny` does not stop sibling hooks from executing. For `PreToolUse`, the most restrictive answer wins: `deny > defer > ask > allow`. Text from `additionalContext` is kept from every hook.

## Related concepts

- [Hook Lifecycle Events](./hook-lifecycle-events.md) — the full event list
- [Hook Types](./hook-types.md) — how each hook runs
- [Hook Exit Codes](./hook-exit-codes.md) — communicating decisions
- [Hook Scope](./hook-scope.md) — where hooks are configured
