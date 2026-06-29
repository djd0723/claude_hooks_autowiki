---
type: concept
title: "Plugin Environment Variables"
created: 2026-06-29
updated: 2026-06-29
tags: [plugins, environment-variables, paths, CLAUDE_PLUGIN_ROOT, CLAUDE_PLUGIN_DATA]
source_count: 1
sources:
  - sources/clean/code-claude-com-docs-en-plugins-reference-md.md
---

# Plugin Environment Variables

Claude Code provides three path variables for referencing locations relative to a plugin's runtime context. All are substituted inline anywhere they appear in skill content, agent content, hook commands, monitor commands, and MCP or LSP server configs. All are also exported as environment variables to hook processes and MCP/LSP server subprocesses.

## `${CLAUDE_PLUGIN_ROOT}`

The absolute path to the plugin's installation directory. Use this to reference scripts, binaries, and config files bundled with the plugin.

**Usage in hook commands (exec form)** — pass via `args` so the path is one argument with no quoting:
```json
{
  "type": "command",
  "args": ["${CLAUDE_PLUGIN_ROOT}/scripts/format-code.sh"]
}
```

**Usage in shell-form hooks and monitor commands** — wrap in double quotes:
```json
{
  "command": "\"${CLAUDE_PLUGIN_ROOT}\"/scripts/format-code.sh"
}
```

**Important**: this path changes when the plugin updates. The previous version's directory remains on disk for about 7 days after an update (for running sessions), but treat it as ephemeral — do not write state here.

When a plugin updates mid-session, hook commands, monitors, MCP servers, and LSP servers keep using the previous version's path. Run `/reload-plugins` to switch hooks, MCP servers, and LSP servers to the new path; monitors require a session restart.

## `${CLAUDE_PLUGIN_DATA}`

A persistent directory for plugin state that survives across updates. Use this for installed dependencies (e.g., `node_modules`, Python virtual environments), generated code, caches, and any files that should persist across plugin versions. The directory is created automatically the first time this variable is referenced.

**Path**: `~/.claude/plugins/data/{id}/` where `{id}` is the plugin identifier with characters outside `a-z`, `A-Z`, `0-9`, `_`, and `-` replaced by `-`. For example, `formatter@my-marketplace` → `~/.claude/plugins/data/formatter-my-marketplace/`.

**Lifecycle**: the data directory is deleted automatically when you uninstall the plugin from the last scope where it is installed. `--keep-data` preserves it (e.g., when reinstalling to test a new version). The `/plugin` interface shows the directory size and prompts before deleting.

### Pattern: install dependencies on first run and on manifest changes

Because the data directory outlives any single plugin version, checking for directory existence alone cannot detect when an update changes the dependency manifest. The recommended pattern compares the bundled manifest against a copy in the data directory:

```json
{
  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "diff -q \"${CLAUDE_PLUGIN_ROOT}/package.json\" \"${CLAUDE_PLUGIN_DATA}/package.json\" >/dev/null 2>&1 || (cd \"${CLAUDE_PLUGIN_DATA}\" && cp \"${CLAUDE_PLUGIN_ROOT}/package.json\" . && npm install) || rm -f \"${CLAUDE_PLUGIN_DATA}/package.json\""
          }
        ]
      }
    ]
  }
}
```

The `diff` exits nonzero when the stored copy is missing or differs, covering both first run and dependency-changing updates. If `npm install` fails, the trailing `rm` removes the copied manifest so the next session retries.

Then reference `node_modules` from an MCP server:
```json
{
  "mcpServers": {
    "routines": {
      "command": "node",
      "args": ["${CLAUDE_PLUGIN_ROOT}/server.js"],
      "env": { "NODE_PATH": "${CLAUDE_PLUGIN_DATA}/node_modules" }
    }
  }
}
```

## `${CLAUDE_PROJECT_DIR}`

The project root — the directory Claude Code was launched from. Equivalent to the `CLAUDE_PROJECT_DIR` variable received by hook processes. Use this to reference project-local scripts or config files.

Wrap in quotes to handle paths with spaces:
```json
{
  "command": "\"${CLAUDE_PROJECT_DIR}/scripts/server.sh\""
}
```

MCP servers can also call the MCP `roots/list` request, which returns the directory Claude Code was launched from.

## Variable substitution contexts

| Context | Variables available |
| :------ | :------------------ |
| Skill content | `${CLAUDE_PLUGIN_ROOT}`, `${CLAUDE_PLUGIN_DATA}`, `${CLAUDE_PROJECT_DIR}`, `${user_config.*}` (non-sensitive only) |
| Agent content | Same as skill content |
| Hook commands | All three path variables + `${user_config.*}` + `${ENV_VAR}` from environment |
| Monitor commands | Same as hook commands |
| MCP server configs | All three path variables + `${user_config.*}` + `${ENV_VAR}` from environment |
| LSP server configs | Same as MCP server configs |

## Related concepts

- [Plugin Manifest Schema](./plugin-manifest-schema.md) — `userConfig` field and `${user_config.*}` substitution
- [Plugin Components](./plugin-components.md) — which components use these variables
- [Plugin Directory Structure](./plugin-directory-structure.md) — what lives in the plugin root vs. data directory
- [Plugin Installation Scopes](./plugin-installation-scopes.md) — caching behavior and path stability
