---
type: concept
title: "Subagent Configuration"
created: 2026-06-29
updated: 2026-06-29
tags: [subagents, configuration, frontmatter, scopes, model]
source_count: 1
sources:
  - sources/clean/code-claude-com-docs-en-sub-agents-md.md
---

# Subagent Configuration

[Subagents](./subagents.md) are defined in Markdown files with YAML frontmatter, followed by the system prompt in Markdown:

```markdown
---
name: code-reviewer
description: Reviews code for quality and best practices
tools: Read, Glob, Grep
model: sonnet
---

You are a code reviewer. When invoked, analyze the code and provide
specific, actionable feedback on quality, security, and best practices.
```

The frontmatter defines metadata and configuration; the body becomes the system prompt. Subagents receive **only this system prompt plus basic environment details** like the working directory, not the full Claude Code system prompt.

> Subagents are loaded at session start. If you add or edit a subagent file directly on disk, restart your session to load it. Subagents created through the `/agents` interface take effect immediately without a restart.

A subagent starts in the main conversation's current working directory. Within a subagent, `cd` commands don't persist between Bash calls and don't affect the main conversation. To give it an isolated copy of the repository, set `isolation: worktree`.

## Where subagents live (scope and precedence)

Store subagent files in different locations depending on scope. When multiple subagents share the same `name`, Claude Code uses the one from the higher-priority location.

| Location | Scope | Priority |
| :------- | :---- | :------- |
| Managed settings | Organization-wide | 1 (highest) |
| `--agents` CLI flag | Current session | 2 |
| `.claude/agents/` | Current project | 3 |
| `~/.claude/agents/` | All your projects | 4 |
| Plugin's `agents/` directory | Where plugin is enabled | 5 (lowest) |

- **Project subagents** (`.claude/agents/`) are ideal for codebase-specific agents; check them into version control. They're discovered by walking up from the working directory, so every `.claude/agents/` between there and the repo root is scanned. As of v2.1.178, when nested directories define the same `name`, the definition closest to the working directory wins. Directories added with `--add-dir` are also scanned.
- **User subagents** (`~/.claude/agents/`) are personal, available in all your projects.
- Both project and user scopes are scanned **recursively**, so you can organize files into subfolders like `agents/review/`. The subfolder path doesn't affect identity — identity comes only from the `name` field. Keep `name` unique across the tree: two files in one scope with the same name means one is kept and the other discarded without warning.
- **Plugin** `agents/` directories are also scanned recursively, but here a subfolder *becomes part of* the [scoped identifier](./subagent-invocation.md): `agents/review/security.md` in plugin `my-plugin` registers as `my-plugin:review:security`. See [Plugin Components](./plugin-components.md).
- **CLI-defined subagents** are passed as JSON via `--agents` when launching Claude Code. They exist only for that session and aren't saved to disk. The flag accepts the same fields as file-based subagents, using `prompt` for the system prompt.
- **Managed subagents** are deployed by administrators as markdown files in `.claude/agents/` inside the [managed settings directory](./settings-files.md), and take precedence over project and user subagents with the same name.

> For security reasons, plugin subagents don't support the `hooks`, `mcpServers`, or `permissionMode` frontmatter fields. These fields are ignored when loading agents from a plugin.

## Supported frontmatter fields

Only `name` and `description` are required.

| Field | Description |
| :---- | :---------- |
| `name` | Unique identifier, lowercase letters and hyphens. [Hooks](./subagent-hooks.md) receive this as `agent_type`. The filename doesn't have to match |
| `description` | When Claude should delegate to this subagent |
| `tools` | [Tools](./subagent-tool-access.md) the subagent can use; inherits all if omitted |
| `disallowedTools` | Tools to deny, removed from the inherited or specified list |
| `model` | `sonnet`, `opus`, `haiku`, `fable`, a full model ID, or `inherit` (the default) |
| `permissionMode` | `default`, `acceptEdits`, `auto`, `dontAsk`, `bypassPermissions`, or `plan` |
| `maxTurns` | Maximum agentic turns before the subagent stops |
| `skills` | Skills to preload into context at startup (full content injected, not just the description) |
| `mcpServers` | [MCP servers](./subagent-tool-access.md) available to this subagent |
| `hooks` | [Lifecycle hooks](./subagent-hooks.md) scoped to this subagent |
| `memory` | Persistent memory scope: `user`, `project`, or `local` |
| `background` | `true` to always run as a [background task](./subagent-invocation.md) (default `false`) |
| `effort` | Effort level while active: `low`, `medium`, `high`, `xhigh`, `max` |
| `isolation` | `worktree` to run in a temporary git worktree branched from your default branch |
| `color` | Display color in the task list and transcript |
| `initialPrompt` | Auto-submitted as the first user turn when run as the main session agent |

## Choosing a model

The `model` field can be an alias (`sonnet`/`opus`/`haiku`/`fable`), a full model ID such as `claude-opus-4-8`, or `inherit`. When omitted it defaults to `inherit`. When Claude invokes a subagent it can also pass a per-invocation `model`. Resolution order:

1. The `CLAUDE_CODE_SUBAGENT_MODEL` environment variable, if set
2. The per-invocation `model` parameter
3. The subagent definition's `model` frontmatter
4. The main conversation's model

The environment variable, per-invocation parameter, and frontmatter values are checked against your organization's `availableModels` allowlist. A value that resolves to an excluded model is not used, and the subagent runs on the inherited model instead.

## Persistent memory

The `memory` field gives the subagent a directory that survives across conversations, building knowledge over time (codebase patterns, debugging insights, architectural decisions):

| Scope | Location | Use when |
| :---- | :------- | :------- |
| `user` | `~/.claude/agent-memory/<name>/` | learnings apply across all projects |
| `project` | `.claude/agent-memory/<name>/` | knowledge is project-specific and shareable via version control |
| `local` | `.claude/agent-memory-local/<name>/` | project-specific but not checked into version control |

`project` is the recommended default. When memory is enabled, the subagent's system prompt gains read/write instructions for the directory plus the first 200 lines or 25KB of `MEMORY.md` (whichever comes first), and Read/Write/Edit are automatically enabled. Include memory instructions directly in the subagent's body so it proactively maintains its own knowledge base.

## The /agents command

The `/agents` command opens a tabbed interface for managing subagents. The **Running** tab lists live and recently finished subagents; the **Library** tab lets you view all subagents (built-in, user, project, plugin), create new ones with guided setup or Claude generation, edit configuration and tool access, delete custom subagents, and see which are active when duplicates exist. This is the recommended way to create and manage subagents.

## Related concepts

- [Subagents](./subagents.md) — what subagents are and when to use them
- [Subagent Tool Access](./subagent-tool-access.md) — `tools`, `disallowedTools`, MCP, and permission modes
- [Subagent Hooks](./subagent-hooks.md) — the `hooks` frontmatter field and lifecycle events
- [Subagent Invocation](./subagent-invocation.md) — scoped identifiers and `--agent` sessions
- [Settings Files](./settings-files.md) — the managed settings directory that hosts managed subagents
