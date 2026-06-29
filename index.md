---
type: index
created: 2026-06-29
updated: 2026-06-29
tags: [hooks, plugins, skills, subagents, permissions, settings, tools, sdk, navigation, index, reference]
---

# Claude Code Hooks & Plugins

Master navigation. Maintained by the indexer. Start with [overview](overview.md)
and the running [synthesis](synthesis.md).

## Sources

- [Hooks Guide](sources/clean/code-claude-com-docs-en-hooks-guide-md.md) — official Claude Code hooks tutorial (clean source)
- [Hooks Reference](sources/clean/code-claude-com-docs-en-hooks-md.md) — official hooks reference: event schemas, JSON I/O, decision control (clean source)
- [Plugins Reference](sources/clean/code-claude-com-docs-en-plugins-reference-md.md) — official plugins reference: components, manifest schema, CLI, scopes (clean source)

## Summaries

- [Hooks Guide](summaries/hooks-guide.md) — automate actions with hooks (official guide)
- [Hooks Reference](summaries/hooks.md) — event schemas, JSON I/O, and decision control (reference doc)
- [Plugins Reference](summaries/plugins-reference.md) — plugin system: components, scopes, manifest, CLI, versioning
- [Create plugins](summaries/plugins.md) — authoring guide: plugin structure, development, distribution
- [Configure permissions](summaries/permissions.md) — permission rules, modes, and evaluation algorithm
- [Extend Claude with skills](summaries/skills.md) — SKILL.md authoring, discovery, invocation, evaluation
- [Create custom subagents](summaries/sub-agents.md) — subagent configuration, context, tool access, hooks
- [Claude Code settings](summaries/settings.md) — settings.json, scopes, precedence, managed and sandbox settings
- [Tools reference](summaries/tools-reference.md) — built-in tool descriptions, Bash model, file behavior
- [Agent SDK hooks](summaries/agent-sdk-hooks.md) — SDK callback hooks, event parity, patterns
- [Claude Code features in the SDK](summaries/agent-sdk-features.md) — settings, skills, and instructions available in SDK agents

## Concepts

### Hooks

- [Hook Lifecycle Events](concepts/hook-lifecycle-events.md) — all 30+ events and when they fire
- [Hook Types](concepts/hook-types.md) — command, http, mcp_tool, prompt, agent
- [Hook Matchers](concepts/hook-matchers.md) — matcher field, `if` field, per-event filter table
- [Hook Exit Codes](concepts/hook-exit-codes.md) — exit 0/2/other and structured JSON output
- [Hook Scope](concepts/hook-scope.md) — configuration location and permission mode interactions
- [Hook Input and Output](concepts/hook-input-output.md) — JSON input envelope and output fields per event
- [Hook Decision Control](concepts/hook-decision-control.md) — per-event decision patterns reference
- [Hook Automation Use Cases](concepts/hook-automation-use-cases.md) — concrete recipes for what to do at each lifecycle point
- [Setup Hooks](concepts/setup-hooks.md) — onboarding and maintenance pattern using the once-per-session Setup event
- [Hooks Adoption Ladder](concepts/hooks-adoption-ladder.md) — practitioner playbook for why and when to adopt hooks
- [Production Hook Patterns](concepts/production-hook-patterns.md) — ClaudeKit case study: design lessons from a shipped hook set
- [StatusLine-Driven Context Backup](concepts/statusline-context-backup.md) — StatusLine-triggered context recovery pattern (alternative to PreCompact)

### Plugins

- [Plugin Components](concepts/plugin-components.md) — skills, agents, hooks, MCP, LSP, monitors, themes
- [Plugin Manifest Schema](concepts/plugin-manifest-schema.md) — plugin.json field reference
- [Plugin Installation Scopes](concepts/plugin-installation-scopes.md) — user/project/local/managed and skills-dir plugins
- [Plugin Directory Structure](concepts/plugin-directory-structure.md) — standard layout and file locations
- [Plugin Environment Variables](concepts/plugin-environment-variables.md) — CLAUDE_PLUGIN_ROOT, CLAUDE_PLUGIN_DATA, CLAUDE_PROJECT_DIR
- [Plugin CLI Commands](concepts/plugin-cli-commands.md) — init, install, uninstall, enable, disable, update, list, validate
- [Plugin Versioning](concepts/plugin-versioning.md) — explicit vs commit-SHA versioning and cache lifecycle
- [Creating Plugins](concepts/creating-plugins.md) — authoring a plugin: structure, components, and init workflow
- [Plugin Default Settings](concepts/plugin-default-settings.md) — shipping a default settings.json inside a plugin
- [Plugin Development Workflow](concepts/plugin-development-workflow.md) — local dev loop: loading, hot-reloading, debugging
- [Plugin Distribution](concepts/plugin-distribution.md) — sharing via private and public marketplaces
- [Plugin Migration](concepts/plugin-migration.md) — converting existing .claude/ skills and hooks into a plugin
- [Plugins vs Standalone Configuration](concepts/plugins-vs-standalone.md) — when to use a plugin vs standalone .claude/ configuration

### Skills

- [Skills](concepts/skills.md) — overview: SKILL.md files extend Claude's toolkit on demand
- [Skill Frontmatter](concepts/skill-frontmatter.md) — frontmatter fields that configure a skill
- [Skill Arguments](concepts/skill-arguments.md) — arguments and built-in substitution variables
- [Skill Invocation Control](concepts/skill-invocation-control.md) — user-only and agent-only invocation restrictions
- [Skill Discovery](concepts/skill-discovery.md) — where to store a skill and who can use it
- [Skill Evaluation](concepts/skill-evaluation.md) — how to verify a skill works beyond just seeing it trigger
- [Skill Content Lifecycle](concepts/skill-content-lifecycle.md) — how skill content enters and persists in the conversation
- [Skill Dynamic Context Injection](concepts/skill-dynamic-context.md) — shell command substitution to inject live data into skill content
- [Skill Subagent Execution](concepts/skill-subagent-execution.md) — `context: fork` runs a skill in an isolated subagent
- [Bundled Skills](concepts/bundled-skills.md) — built-in skills shipped with every Claude Code session

### Subagents

- [Subagents](concepts/subagents.md) — overview: specialized AI assistants for specific task types
- [Subagent Configuration](concepts/subagent-configuration.md) — YAML frontmatter and system prompt authoring
- [Subagent Invocation](concepts/subagent-invocation.md) — automatic delegation vs. explicit user invocation
- [Subagent Tool Access](concepts/subagent-tool-access.md) — controlling tool access, modes, and conditional rules
- [Subagent Context](concepts/subagent-context.md) — fresh isolated context window per subagent
- [Subagent Hooks](concepts/subagent-hooks.md) — hooks scoped to subagent lifecycle
- [Forked Subagents](concepts/forked-subagents.md) — fork inherits parent conversation instead of starting fresh

### Permissions

- [Permission Evaluation](concepts/permission-evaluation.md) — how tool calls are resolved: allow, ask, or deny
- [Permission Modes](concepts/permission-modes.md) — session-wide presets before individual rules are consulted
- [Permission Settings](concepts/permission-settings.md) — the permissions block in settings.json
- [Tool Permission Rules](concepts/tool-permission-rules.md) — ToolName(specifier) rule format for each built-in tool
- [Bash Permission Matching](concepts/bash-permission-matching.md) — Bash's special matching mechanics for safe gating
- [File Permission Patterns](concepts/file-permission-patterns.md) — gitignore-style path patterns for Read/Edit/Cd rules
- [AI Permission Reviewer](concepts/ai-permission-reviewer.md) — ClaudeFast Permission Hook case study: approve/deny via hook

### Settings

- [Settings Files](concepts/settings-files.md) — settings.json: the official Claude Code configuration mechanism
- [Configuration Scopes](concepts/configuration-scopes.md) — scope system: who a setting is shared with
- [Settings Precedence](concepts/settings-precedence.md) — how conflicts between scopes are resolved
- [Managed Settings](concepts/managed-settings.md) — admin-deployed settings that can't be overridden
- [Sandbox Settings](concepts/sandbox-settings.md) — OS-level bash sandboxing on macOS/Linux/WSL2

### Tools

- [Built-in Tools](concepts/built-in-tools.md) — full list of tools Claude Code has access to
- [Bash Execution Model](concepts/bash-execution-model.md) — each command runs in a separate process; what persists
- [File Tool Behavior](concepts/file-tool-behavior.md) — Read/Edit/Write/Glob/Grep behavioral details
- [Working Directories](concepts/working-directories.md) — extending file access vs. relocating the session root

### Agent SDK

- [SDK Callback Hooks](concepts/sdk-callback-hooks.md) — in-process hooks as callback functions, not shell commands
- [SDK Hook Event Availability](concepts/sdk-hook-events.md) — Python vs. TypeScript hook parity reference
- [SDK Hook Patterns](concepts/sdk-hook-patterns.md) — common PreToolUse patterns and failure modes
- [SDK Setting Sources](concepts/sdk-setting-sources.md) — SDK agents share the same settings foundation as Claude Code
- [SDK Skills Loading](concepts/sdk-skills-loading.md) — skills load on demand in SDK agents
- [SDK Project Instructions](concepts/sdk-project-instructions.md) — CLAUDE.md and rules files in SDK agents

## Entities

## Comparisons

- [Hooks Guide accuracy check](comparisons/code-claude-com-docs-en-hooks-guide-md.md) — verified wiki pages against source; 2 inaccuracies corrected
- [matcher vs if field](comparisons/matcher-vs-if-field.md) — hook targeting levels: group-level name vs argument-level filtering
- [Sync vs async hooks](comparisons/sync-vs-async-hooks.md) — synchronous vs async: true vs asyncRewake: true execution modes
- [Plugin scopes vs hook scopes](comparisons/plugin-scopes-vs-hook-scopes.md) — same four-tier hierarchy, different capability restrictions
- [Monitors vs command hooks](comparisons/monitors-vs-command-hooks.md) — persistent background stream vs event-triggered single-shot
- [Skills-dir plugins vs marketplace plugins](comparisons/skills-dir-plugins-vs-marketplace-plugins.md) — in-place discovery vs cache copy; path and capability differences
- [Permission rules vs. permission hooks](comparisons/permission-rules-vs-permission-hooks.md) — static rules vs. hook judgment for tool access control
- [Choosing an SDK extension feature](comparisons/sdk-extension-features.md) — which SDK extension mechanism fits which use case
- [Description-driven activation vs. activation hook](comparisons/description-driven-activation-vs-activation-hook.md) — model-judgment description matching vs. deterministic UserPromptSubmit hook; when each wins and how they layer
- [CLAUDE.md rules vs. skills workflows](comparisons/claude-md-rules-vs-skills-workflows.md) — always-on rules vs. on-demand workflows; the symmetric "pay twice" failure and three-way decision tree

## Answers
