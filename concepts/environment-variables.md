---
type: concept
title: "Environment Variables (Daily-Development Subset)"
created: 2026-06-29
updated: 2026-06-29
tags: [settings, environment-variables, configuration, models, context]
source_count: 1
sources:
  - sources/clean/claudefa-st-blog-guide-configuration-basics.md
---

# Environment Variables (Daily-Development Subset)

The [settings reference](../summaries/settings.md) notes that "any environment variable can be set under the `env` key" and defers the full catalog (~70 variables) to the official env-vars reference. This page captures the practitioner-curated *subset that matters for daily development* — the variables worth memorizing — mined from one config walkthrough.

> "Environment variables give you fine-grained control over Claude Code behavior. You can set them in your shell profile, pass them inline, or add them to the env key in settings.json."

> "For the full list of ~70 environment variables, see the settings reference."

## The essential variables

| Variable | What it does |
| :------- | :----------- |
| `ANTHROPIC_MODEL` | Set the default model (e.g., `claude-sonnet-4-20250514`) |
| `CLAUDE_CODE_MAX_OUTPUT_TOKENS` | Max output tokens per response. Default: 32,000. Max: 64,000 |
| `MAX_THINKING_TOKENS` | Control the extended thinking budget. Default: 31,999. Set to 0 to disable |
| `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` | Trigger auto-compaction at a specific context % (1–100) |
| `CLAUDE_CODE_SUBAGENT_MODEL` | Set the model for sub-agents separately from your main model |
| `BASH_DEFAULT_TIMEOUT_MS` | Default timeout for long-running bash commands |
| `BASH_MAX_OUTPUT_LENGTH` | Max characters in bash output before truncation |
| `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` | Disable auto-updates, telemetry, error reporting, and bug reports in one variable |
| `DISABLE_AUTOUPDATER` | Disable automatic updates only (set to 1) |
| `CLAUDE_CODE_ENABLE_TASKS` | Set to false to revert to the previous TODO list system |
| `MAX_MCP_OUTPUT_TOKENS` | Max tokens in MCP tool responses (default: 25,000) |
| `MCP_TIMEOUT` | Timeout in ms for MCP server startup |
| `CLAUDE_CODE_SHELL` | Override automatic shell detection |
| `CLAUDE_CONFIG_DIR` | Store configuration and data files in a custom directory |
| `HTTP_PROXY` / `HTTPS_PROXY` | Route connections through a proxy server |

## Three ways to set them

The same variable can be applied at three different lifetimes, trading reach for persistence:

```sh
# Single session — prefix the command
ANTHROPIC_MODEL=claude-sonnet-4-20250514 claude

# Persistent for your shell — add to ~/.bashrc or ~/.zshrc
export CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=80
export MAX_THINKING_TOKENS=10000
```

Or set them once in `settings.json` so they apply automatically to every session (and propagate to spawned subprocesses — see the `env` key in the [settings reference](../summaries/settings.md)):

```json
{
  "env": {
    "CLAUDE_AUTOCOMPACT_PCT_OVERRIDE": "80",
    "MAX_THINKING_TOKENS": "10000"
  }
}
```

Setting a variable under the `env` key is the [scoped](./configuration-scopes.md) form: a user-scope `settings.json` makes it follow you across projects, a project-scope file shares it with collaborators.

## Related concepts

- [Configuration Scopes](./configuration-scopes.md) — where the `env` key lives in the four-scope hierarchy
- [Settings Files](./settings-files.md) — the files the `env` block is written into and how they reload
- [Settings reference](../summaries/settings.md) — the `env` key, the note that many settings have equivalent env vars, and the pointer to the full ~70-variable catalog
- [Subagent Configuration](./subagent-configuration.md) — `CLAUDE_CODE_SUBAGENT_MODEL` sets the sub-agent model independently of the main model
