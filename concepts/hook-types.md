---
type: concept
title: "Hook Types"
created: 2026-06-29
updated: 2026-06-29
tags: [hooks, command, prompt, agent, http, mcp]
source_count: 2
sources:
  - sources/clean/code-claude-com-docs-en-hooks-guide-md.md
  - sources/clean/code-claude-com-docs-en-hooks-md.md
---

# Hook Types

Each hook has a `type` field that determines how it runs. There are five types.

## `"type": "command"` — shell command (default)

Runs a shell command. The event's JSON data arrives on stdin; the hook communicates back via stdout, stderr, and exit code.

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "jq -r '.tool_input.file_path' | xargs npx prettier --write"
          }
        ]
      }
    ]
  }
}
```

Default timeout: 10 minutes. `UserPromptSubmit` lowers this to 30 seconds; `MessageDisplay` to 10 seconds.

### Exec form vs shell form

A command hook runs in **shell form** (default) when `args` is omitted — the `command` string is passed to `sh -c` on macOS/Linux, allowing pipes, `&&`, and shell features.

It runs in **exec form** when `args` is present — `command` is resolved as an executable and spawned directly with `args` as the argument vector, with no shell involved. This is the right choice when referencing path placeholders, since each element is passed as one argument without quoting.

```json
{
  "type": "command",
  "command": "node",
  "args": ["${CLAUDE_PLUGIN_ROOT}/scripts/format.js", "--fix"]
}
```

### Path placeholders

Use these inside `command` and `args` to reference scripts without hardcoded paths:

| Placeholder | Resolves to |
| :---------- | :---------- |
| `${CLAUDE_PROJECT_DIR}` | The project root |
| `${CLAUDE_PLUGIN_ROOT}` | The plugin's installation directory (changes on plugin updates) |
| `${CLAUDE_PLUGIN_DATA}` | The plugin's persistent data directory (survives updates) |

Both forms also export these as environment variables on the spawned process.

### Async command fields

| Field | Description |
| :---- | :---------- |
| `async: true` | Run in the background. Decision output fields (`decision`, `permissionDecision`, `continue`) are ignored — the action has already completed |
| `asyncRewake: true` | Run in the background and wake Claude on exit code 2. Claude receives the hook's stderr (or stdout if stderr is empty) as a system reminder, so it can react to a long-running background failure |

## `"type": "http"` — POST to an endpoint

POSTs event data to an HTTP endpoint. The endpoint receives the same JSON that a command hook would receive on stdin and returns results through the HTTP response body.

> "HTTP hooks are useful when you want a web server, cloud function, or external service to handle hook logic: for example, a shared audit service that logs tool use events across a team."

Header values support `$VAR_NAME` or `${VAR_NAME}` interpolation. Only variables listed in `allowedEnvVars` are resolved.

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "hooks": [
          {
            "type": "http",
            "url": "http://localhost:8080/hooks/tool-use",
            "headers": { "Authorization": "Bearer $MY_TOKEN" },
            "allowedEnvVars": ["MY_TOKEN"]
          }
        ]
      }
    ]
  }
}
```

HTTP status codes alone cannot block actions — you must return appropriate JSON in the response body. Non-2xx responses, connection failures, and timeouts all produce non-blocking errors that allow execution to continue. To block, return a 2xx response with `decision: "block"` or `hookSpecificOutput.permissionDecision: "deny"` in the body.

Additional fields beyond common ones:

| Field | Description |
| :---- | :---------- |
| `url` | URL to POST to |
| `headers` | Additional HTTP headers; values support `$VAR` or `${VAR}` interpolation |
| `allowedEnvVars` | Variables allowed to be interpolated into header values |

## `"type": "mcp_tool"` — call a connected MCP server tool

Calls a tool on an already-connected MCP server. See the Hooks reference for full configuration.

## `"type": "prompt"` — single-turn LLM evaluation

Sends your prompt and the hook input to a Claude model (Haiku by default). The model returns a yes/no decision as JSON:

- `"ok": true` — action proceeds
- `"ok": false` — what happens depends on the event:
  - `Stop` / `SubagentStop`: the `reason` feeds back to Claude so it keeps working
  - `PreToolUse`: tool call is denied, `reason` returned to Claude as tool error
  - `PostToolUse`, `PostToolBatch`, `UserPromptSubmit`, `UserPromptExpansion`: turn ends, `reason` appears as warning

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "prompt",
            "prompt": "Check if all tasks are complete. If not, respond with {\"ok\": false, \"reason\": \"what remains to be done\"}."
          }
        ]
      }
    ]
  }
}
```

Timeout: 30 seconds. Specify a different model with the `model` field.

## `"type": "agent"` — multi-turn subagent (experimental)

Spawns a subagent with tool access to verify conditions. Uses the same `"ok"` / `"reason"` response format as prompt hooks, but can read files, search code, and run commands.

> "Use prompt hooks when the hook input data alone is enough to make a decision. Use agent hooks when you need to verify something against the actual state of the codebase."

Default timeout: 60 seconds; up to 50 tool-use turns.

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "agent",
            "prompt": "Verify that all unit tests pass. Run the test suite and check the results.",
            "timeout": 120
          }
        ]
      }
    ]
  }
}
```

Agent hooks are experimental and may change. For production workflows, prefer command hooks.

## Related concepts

- [Hook Lifecycle Events](./hook-lifecycle-events.md) — which events hooks attach to
- [Hook Exit Codes](./hook-exit-codes.md) — how command hooks return decisions
- [Hook Matchers](./hook-matchers.md) — narrowing when hooks fire
- [Hook Input and Output](./hook-input-output.md) — JSON input/output protocol
- [Hook Decision Control](./hook-decision-control.md) — per-event decision patterns
