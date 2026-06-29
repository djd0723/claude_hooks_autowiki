---
type: concept
title: "Hook Scope and Configuration Location"
created: 2026-06-29
updated: 2026-06-29
tags: [hooks, configuration, settings, scope, plugins, skills]
source_count: 2
sources:
  - sources/clean/code-claude-com-docs-en-hooks-guide-md.md
  - sources/clean/code-claude-com-docs-en-hooks-md.md
---

# Hook Scope and Configuration Location

Where you add a hook determines its scope.

## Scope table

| Location | Scope | Shareable |
| :-------- | :---- | :-------- |
| `~/.claude/settings.json` | All your projects | No, local to your machine |
| `.claude/settings.json` | Single project | Yes, can be committed to the repo |
| `.claude/settings.local.json` | Single project | No, gitignored when Claude Code creates it |
| Managed policy settings | Organization-wide | Yes, admin-controlled |
| Plugin `hooks/hooks.json` | When plugin is enabled | Yes, bundled with the plugin |
| Skill or agent frontmatter | While the skill or agent is active | Yes, defined in the component file |

## Configuration structure

All hooks live inside a `hooks` object in the settings file. Each key is a hook event name; its value is an array of hook groups:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          { "type": "command", "command": "..." }
        ]
      }
    ],
    "Notification": [
      {
        "matcher": "",
        "hooks": [{ "type": "command", "command": "..." }]
      }
    ]
  }
}
```

When adding a new event to an existing `hooks` key, add the new event name as a sibling — don't replace the entire object.

## Managing hooks

- Run `/hooks` in Claude Code to browse all configured hooks grouped by event. The menu shows all five hook types (`command`, `prompt`, `agent`, `http`, `mcp_tool`) with a `[type]` prefix and labels their source (`User`, `Project`, `Local`, `Plugin`, `Session`, `Built-in`). The menu is read-only; edit settings JSON directly or ask Claude to make changes.
- To disable all hooks: set `"disableAllHooks": true` in your settings file. Managed hooks cannot be disabled this way — only `disableAllHooks` in managed settings itself can disable managed hooks.
- If you edit settings files while Claude Code is running, the file watcher normally picks up changes automatically.

## Enterprise: `allowManagedHooksOnly`

Enterprise administrators can set `allowManagedHooksOnly: true` to block all user, project, and plugin hooks. Plugins force-enabled in managed settings `enabledPlugins` are exempt, allowing vetted hooks to be distributed via an organization marketplace.

## Hooks in skills and agents

Hooks declared in skill or agent frontmatter are scoped to the component's lifecycle. All hook events are supported. For subagents, `Stop` hooks are automatically converted to `SubagentStop`. The `once` field (honored only in skill frontmatter) makes a hook run once per session and then remove itself:

```yaml
hooks:
  SessionStart:
    - hooks:
        - type: command
          command: ./scripts/setup.sh
          once: true
```

The `once` field is ignored in settings files and agent frontmatter.

## Hook scripts

For complex hooks, save logic to a script file and reference it:

```json
{
  "type": "command",
  "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/my-hook.sh"
}
```

Scripts must be executable (`chmod +x`). The `$CLAUDE_PROJECT_DIR` variable resolves to the project root, avoiding path issues.

To skip the shell entirely (exec form), add `"args": []` — this spawns the script directly without `sh -c`.

## Permission mode interaction

- `PreToolUse` hooks fire **before** any permission-mode check. A hook returning `deny` blocks even in `bypassPermissions` mode.
- A hook returning `allow` does not bypass deny rules from settings. Hooks can tighten restrictions but not loosen them past what permission rules allow.
- `PermissionRequest` hooks do not fire in non-interactive mode (`-p`); use `PreToolUse` instead.

## Related concepts

- [Hook Lifecycle Events](./hook-lifecycle-events.md) — the full event list
- [Hook Types](./hook-types.md) — command, prompt, agent, http, mcp_tool
- [Hook Matchers](./hook-matchers.md) — filtering which hooks fire
- [Hook Exit Codes](./hook-exit-codes.md) — communicating decisions
