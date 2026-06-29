---
type: concept
title: "Plugin Manifest Schema"
created: 2026-06-29
updated: 2026-06-29
tags: [plugins, manifest, schema, plugin-json, configuration]
source_count: 1
sources:
  - sources/clean/code-claude-com-docs-en-plugins-reference-md.md
---

# Plugin Manifest Schema

The `.claude-plugin/plugin.json` file is the plugin manifest. It is **optional** — Claude Code auto-discovers components in default locations and uses the directory name as the plugin name when no manifest exists. Use a manifest when you need stable naming, metadata, or custom component paths.

## Minimum valid manifest

```json
{
  "name": "my-plugin"
}
```

`name` is the only required field when a manifest is present. It must be kebab-case with no spaces, and is used for namespacing all components (e.g., agent `agent-creator` in plugin `plugin-dev` appears as `plugin-dev:agent-creator`).

## Complete schema

```json
{
  "name": "plugin-name",
  "displayName": "Plugin Name",
  "version": "1.2.0",
  "description": "Brief plugin description",
  "author": { "name": "Author Name", "email": "author@example.com", "url": "https://github.com/author" },
  "homepage": "https://docs.example.com/plugin",
  "repository": "https://github.com/author/plugin",
  "license": "MIT",
  "keywords": ["keyword1", "keyword2"],
  "defaultEnabled": false,
  "skills": "./custom/skills/",
  "commands": ["./custom/commands/special.md"],
  "agents": ["./custom/agents/reviewer.md"],
  "hooks": "./config/hooks.json",
  "mcpServers": "./mcp-config.json",
  "outputStyles": "./styles/",
  "lspServers": "./.lsp.json",
  "experimental": {
    "themes": "./themes/",
    "monitors": "./monitors.json"
  },
  "userConfig": { ... },
  "channels": [ ... ],
  "dependencies": [
    "helper-lib",
    { "name": "secrets-vault", "version": "~2.1.0" }
  ]
}
```

## Metadata fields

| Field | Description |
| :---- | :---------- |
| `name` | **Required.** Unique identifier (kebab-case). Used for namespacing and lookup. |
| `displayName` | Human-readable name for UI surfaces (spaces allowed). Falls back to `name`. Requires v2.1.143+. |
| `version` | Semantic version string. Controls update behavior. See [Plugin Versioning](./plugin-versioning.md). |
| `description` | Brief explanation of purpose. |
| `author` | Object with `name`, `email`, and `url` fields. |
| `homepage` | Documentation URL. |
| `repository` | Source code URL. |
| `license` | License identifier (e.g., `"MIT"`, `"Apache-2.0"`). |
| `keywords` | Array of discovery tags. |
| `defaultEnabled` | Whether the plugin starts enabled when the user hasn't set a preference. Defaults to `true`. Requires v2.1.154+. |

## Component path fields

| Field | Type | Behavior |
| :---- | :--- | :------- |
| `skills` | string\|array | **Adds to** the default `skills/` scan. |
| `commands` | string\|array | **Replaces** the default `commands/` directory. |
| `agents` | string\|array | **Replaces** the default `agents/` directory. |
| `hooks` | string\|array\|object | Multiple sources merge; see hooks merge rules. |
| `mcpServers` | string\|array\|object | Multiple sources merge; see MCP merge rules. |
| `outputStyles` | string\|array | **Replaces** the default `output-styles/` directory. |
| `lspServers` | string\|array\|object | Multiple sources merge; see LSP merge rules. |
| `experimental.themes` | string\|array | **Replaces** the default `themes/` directory. |
| `experimental.monitors` | string\|array | **Replaces** the default `monitors/monitors.json`. |

All component paths must be relative to the plugin root and start with `./`. To keep the default directory **and** add more, list it explicitly: `"commands": ["./commands/", "./extras/"]`.

When a manifest key exists alongside a default folder, Claude Code v2.1.140+ flags the ignored default folder in `/doctor` and `claude plugin list`.

## `defaultEnabled`

Set `defaultEnabled: false` to ship a plugin that installs disabled. Users turn it on with `claude plugin enable <plugin>` or the `/plugin` interface. Two things override it:

- **User's explicit setting**: an `enabledPlugins` entry at any scope. Once set, it persists across updates and reinstalls.
- **Dependency requirement**: when another active plugin requires this one, Claude Code writes `true` for it, giving it an explicit setting.

## `userConfig`

Declares values prompted at enable time, avoiding the need for users to hand-edit `settings.json`.

```json
{
  "userConfig": {
    "api_endpoint": {
      "type": "string",
      "title": "API endpoint",
      "description": "Your team's API endpoint"
    },
    "api_token": {
      "type": "string",
      "title": "API token",
      "description": "API authentication token",
      "sensitive": true
    }
  }
}
```

Each key's config fields:

| Field | Required | Description |
| :---- | :------- | :---------- |
| `type` | Yes | `string`, `number`, `boolean`, `directory`, or `file` |
| `title` | Yes | Label shown in the configuration dialog |
| `description` | Yes | Help text beneath the field |
| `sensitive` | No | Masks input; stores in system keychain instead of `settings.json` |
| `required` | No | Validation fails when empty |
| `default` | No | Value used when user provides nothing |
| `multiple` | No | For `string` type, allow an array of strings |
| `min` / `max` | No | Bounds for `number` type |

Values are available as `${user_config.KEY}` in hook commands, MCP/LSP server configs, and monitor commands. Non-sensitive values can also be substituted in skill and agent content. All values are exported as `CLAUDE_PLUGIN_OPTION_<KEY>` environment variables.

Non-sensitive values stored in `settings.json` under `pluginConfigs[<plugin-id>].options`. Sensitive values go to the system keychain (approximately 2 KB total limit shared with OAuth tokens).

## `channels`

Declares message channels that inject content into the conversation. Each channel binds to an MCP server the plugin provides.

```json
{
  "channels": [
    {
      "server": "telegram",
      "userConfig": {
        "bot_token": { "type": "string", "title": "Bot token", "sensitive": true },
        "owner_id": { "type": "string", "title": "Owner ID", "description": "Your Telegram user ID" }
      }
    }
  ]
}
```

The `server` field must match a key in `mcpServers`. The per-channel `userConfig` uses the same schema as the top-level field.

## Unrecognized fields

Claude Code ignores top-level fields it does not recognize. Fields with the wrong **type** (e.g., `keywords` as a string instead of an array) still fail as load errors. `claude plugin validate` reports unrecognized fields as warnings, not errors, and suggests likely intended names for near-matches. Use `--strict` to treat warnings as errors in CI.

## Related concepts

- [Plugin Components](./plugin-components.md) — what each component type does
- [Plugin Directory Structure](./plugin-directory-structure.md) — default file locations and standard layout
- [Plugin Installation Scopes](./plugin-installation-scopes.md) — user/project/local/managed scopes
- [Plugin Versioning](./plugin-versioning.md) — how the `version` field affects update behavior
- [Plugin Environment Variables](./plugin-environment-variables.md) — variable substitution in manifest paths
