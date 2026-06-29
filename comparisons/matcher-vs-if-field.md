---
type: comparison
title: "matcher field vs if field — hook targeting levels"
created: 2026-06-29
updated: 2026-06-29
tags: [hooks, matchers, filtering, configuration, if-field]
sources:
  - sources/clean/code-claude-com-docs-en-hooks-md.md
  - sources/clean/code-claude-com-docs-en-hooks-guide-md.md
---

# `matcher` vs `if` field — hook targeting levels

Both fields narrow which occurrences of an event trigger a hook, but they operate at different levels of granularity. `matcher` selects at the event or tool-name level; `if` selects at the argument level. Understanding the distinction prevents hooks from firing too broadly or not at all.

## Comparison

| Dimension | `matcher` | `if` |
| :-------- | :-------- | :--- |
| Applies to | Hook group (`hooks: [{ matcher: "...", hooks: [...] }]`) | Individual hook within the group |
| Filters on | Tool name, session reason, notification type, agent type, etc. — depends on the event | Tool name **and** argument content together |
| Events supported | Most events (see [Hook Matchers](../concepts/hook-matchers.md) for per-event table) | Tool events only: `PreToolUse`, `PostToolUse`, `PostToolUseFailure`, `PermissionRequest`, `PermissionDenied` |
| Syntax | Exact string, `\|`-separated list, or JavaScript regex | Permission rule syntax: `ToolName(arg pattern)` |
| Fail behavior | No matcher = matches all occurrences | Fails open — runs the hook when arguments cannot be parsed |
| Version required | All versions | Claude Code v2.1.85+ |
| Use `*` for "all" | Yes — `"*"` or omit entirely | Not applicable; `if` is optional and hook runs for all args when absent |

## Example: combining both fields

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
            "command": ".claude/hooks/check-git-policy.sh"
          }
        ]
      }
    ]
  }
}
```

`matcher: "Bash"` — fires the group only on Bash tool calls (not Edit, not Write).
`if: "Bash(git *)"` — within those Bash calls, fires this hook only when a `git` subcommand is present.

Because `if` fails open, a compound command like `echo $(date)` with a pattern that checks more than the command name may still trigger the hook. This is intentional — prefer hook-level safety — but it means `if` is not a reliable hard filter for security decisions. Use the [permission system](../concepts/hook-scope.md) for hard allow/deny enforcement.

## When to use which

**Use `matcher` alone** when any occurrence of a named tool (or event subtype) should trigger the hook:
```json
{ "matcher": "Edit|Write", "hooks": [{ "type": "command", "command": "run-formatter.sh" }] }
```

**Add `if`** when you need to target a specific invocation pattern within a tool — e.g., format only Python files, or check only `git push` calls:
```json
{ "if": "Edit(*.py)", "type": "command", "command": "black $TOOL_ARG_FILE_PATH" }
```

**Combine both** to keep the group efficiently filtered (matcher skips the group for irrelevant tools) while targeting argument patterns inside it (if further narrows within the group).

Do not use `if` on non-tool events (`SessionStart`, `Stop`, etc.) — the hook will silently not run.

## Related concepts

- [Hook Matchers](../concepts/hook-matchers.md) — full matcher and `if` field documentation
- [Hook Decision Control](../concepts/hook-decision-control.md) — per-event decision patterns
- [Hook Scope](../concepts/hook-scope.md) — configuration locations and permission mode
