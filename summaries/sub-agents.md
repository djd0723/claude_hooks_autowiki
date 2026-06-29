---
type: summary
title: "Subagents — configuration, context, and invocation"
slug: code-claude-com-docs-en-sub-agents-md
created: 2026-06-29
updated: 2026-06-29
tags: [subagents, configuration, context, invocation, delegation, tool-access, hooks, orchestration]
source_count: 2
sources:
  - sources/clean/code-claude-com-docs-en-sub-agents-md.md
  - sources/clean/claudefa-st-blog-guide-development-agent-manager-role.md
---

# Summary: Create custom subagents

The authoritative reference for defining, configuring, and invoking subagents in Claude Code — specialized AI assistants that run in their own context window with custom prompts, tool access, and permissions, returning only a summary to the main conversation.

## Key thesis from source

> "Each subagent runs in its own context window with a custom system prompt, specific tool access, and independent permissions. When Claude encounters a task that matches a subagent's description, it delegates to that subagent, which works independently and returns results."

## What this source covers

- What [subagents](../concepts/subagents.md) are and the built-in set (Explore, Plan, general-purpose, and helper agents)
- The agent definition file: Markdown body + YAML frontmatter, scopes, and all [configuration fields](../concepts/subagent-configuration.md)
- [Tool access](../concepts/subagent-tool-access.md): `tools` allowlist, `disallowedTools` denylist, MCP scoping, permission modes
- [Separate context windows](../concepts/subagent-context.md): what loads at startup, resume, auto-compaction
- How and when subagents are [invoked](../concepts/subagent-invocation.md): automatic delegation, explicit naming, @-mention, session-wide `--agent`
- Model selection and resolution order
- [Subagent-specific hooks](../concepts/subagent-hooks.md): frontmatter hooks and project-level `SubagentStart`/`SubagentStop`
- [Forking the current conversation](../concepts/forked-subagents.md) as an inheriting subagent

## What subagents are and when to use them

A subagent does side work in its own context and returns only the summary, keeping search results, logs, and verbose output out of the main conversation. Define a custom one when you keep spawning the same kind of worker with the same instructions. Subagents help you:

- **Preserve context** by keeping exploration and implementation out of the main conversation
- **Enforce constraints** by limiting which tools a subagent can use
- **Reuse configurations** across projects with user-level subagents
- **Specialize behavior** with focused system prompts
- **Control costs** by routing tasks to faster, cheaper models like Haiku

Subagents work within a single session. Claude uses each subagent's `description` to decide when to delegate.

## Built-in subagents

Each inherits the parent conversation's permissions with additional tool restrictions. Explore and Plan skip CLAUDE.md files and git status to stay fast; every other agent loads both.

| Agent | Model | Tools | Purpose |
| :---- | :---- | :---- | :------ |
| Explore | Haiku | read-only (Write/Edit denied) | file discovery, code search, codebase exploration |
| Plan | inherits | read-only (Write/Edit denied) | codebase research during plan mode |
| general-purpose | inherits | all tools | complex, multi-step exploration + modification |
| statusline-setup | Sonnet | — | invoked by `/statusline` |
| claude-code-guide | Haiku | — | answers about Claude Code features |

Built-ins are always registered in interactive sessions. To restrict: add a type to `permissions.deny`, deny the `Agent` tool entirely, or set `CLAUDE_AGENT_SDK_DISABLE_BUILTIN_AGENTS=1` in headless/SDK mode to supply only your own.

## Subagent scopes

Subagents are Markdown files with YAML frontmatter. When names collide, the higher-priority location wins.

| Location | Scope | Priority |
| :------- | :---- | :------- |
| Managed settings | Organization-wide | 1 (highest) |
| `--agents` CLI flag | Current session | 2 |
| `.claude/agents/` | Current project | 3 |
| `~/.claude/agents/` | All your projects | 4 |
| Plugin's `agents/` directory | Where plugin is enabled | 5 (lowest) |

Project and user directories are scanned recursively (subfolders allowed); identity comes only from the `name` field, so names must be unique across the tree. Plugin subfolders, by contrast, become part of the scoped identifier (`my-plugin:review:security`). Project subagents are discovered by walking up from the cwd; as of v2.1.178 the definition closest to the working directory wins among nested duplicates. CLI-defined subagents (`--agents`) accept JSON with the same frontmatter fields and exist only for that session.

> "For security reasons, plugin subagents don't support the `hooks`, `mcpServers`, or `permissionMode` frontmatter fields. These fields are ignored when loading agents from a plugin."

## Supported frontmatter fields

Only `name` and `description` are required. The Markdown body becomes the system prompt.

| Field | Required | Purpose |
| :---- | :------- | :------ |
| `name` | Yes | Unique lowercase-hyphen identifier; hooks receive it as `agent_type` |
| `description` | Yes | When Claude should delegate to this subagent |
| `tools` | No | Allowlist; inherits all if omitted |
| `disallowedTools` | No | Denylist, applied before `tools` |
| `model` | No | `sonnet`/`opus`/`haiku`/`fable`, full ID, or `inherit` (default) |
| `permissionMode` | No | `default`/`acceptEdits`/`auto`/`dontAsk`/`bypassPermissions`/`plan` |
| `maxTurns` | No | Maximum agentic turns before stopping |
| `skills` | No | Skills to preload (full content injected at startup) |
| `mcpServers` | No | MCP servers scoped to this subagent (inline or by name) |
| `hooks` | No | Lifecycle hooks scoped to this subagent |
| `memory` | No | Persistent memory scope: `user`/`project`/`local` |
| `background` | No | `true` to always run as a background task |
| `effort` | No | `low`/`medium`/`high`/`xhigh`/`max`; overrides session effort |
| `isolation` | No | `worktree` for an isolated git worktree copy |
| `color` | No | Display color in task list/transcript |
| `initialPrompt` | No | Auto-submitted first turn when run as main session agent |

> "Subagents receive only this system prompt plus basic environment details like the working directory, not the full Claude Code system prompt."

Files added on disk load at session start (restart to pick up); changes via the `/agents` interface take effect immediately.

## Tool access and capability control

Subagents inherit internal and MCP tools from the main conversation by default. Some UI/session-dependent tools (`AskUserQuestion`, `EnterPlanMode`, `ExitPlanMode` unless `permissionMode: plan`, `ScheduleWakeup`, `WaitForMcpServers`) are never available.

- Restrict with `tools` (allowlist) or `disallowedTools` (denylist). If both are set, `disallowedTools` applies first; a tool in both is removed.
- Both fields accept MCP patterns: `mcp__<server>` / `mcp__<server>__*`; in `disallowedTools`, `mcp__*` removes all MCP tools.
- `mcpServers` can scope servers to a subagent — inline definitions connect at start and disconnect at finish; string references reuse the parent connection. Defining a server inline keeps its tool descriptions out of the main conversation's context.
- `Agent(agent_type)` syntax in `tools` restricts which subagent types a main-thread `--agent` can spawn; in a subagent definition the type list inside parentheses is ignored. Omitting `Agent` entirely prevents spawning any.
- `PreToolUse` hooks give finer conditional control (e.g., allow read-only SQL but block writes by exiting code 2).
- Disable specific subagents via `permissions.deny` with `Agent(name)` or `--disallowedTools "Agent(name)"`.

### Permission modes

| Mode | Behavior |
| :--- | :------- |
| `default` | Standard prompts |
| `acceptEdits` | Auto-accept edits + common filesystem commands in working dir |
| `auto` | Background classifier reviews commands and protected writes |
| `dontAsk` | Auto-deny prompts (explicit allows still work) |
| `bypassPermissions` | Skip prompts (use with caution) |
| `plan` | Read-only exploration |

A parent's `bypassPermissions` or `acceptEdits` takes precedence and can't be overridden; under parent auto mode, the subagent's `permissionMode` is ignored.

## Separate context windows

> "Each subagent starts with a fresh, isolated context window. It doesn't see your conversation history, the skills you've already invoked, or the files Claude has already read."

A non-fork subagent's initial context contains: its own system prompt + environment details, the delegation task message, CLAUDE.md and the full memory hierarchy (Explore and Plan skip this), a git-status snapshot from the parent session start (Explore/Plan skip it; absent outside a repo or when `includeGitInstructions: false`), and any preloaded skills. A fork is the exception — it inherits the parent conversation.

**Resume**: each invocation is a new instance, but Claude can resume a subagent via `SendMessage` with its agent ID — retaining full history. Explore and Plan are one-shot and return no ID. Transcripts persist separately (`~/.claude/projects/{project}/{sessionId}/subagents/agent-{agentId}.jsonl`), survive main-conversation compaction, and are cleaned up per `cleanupPeriodDays` (default 30 days). Subagents also support auto-compaction using the same logic as the main conversation.

## Model selection

The `model` field accepts an alias, full model ID, `inherit`, or omitted (defaults to `inherit`). When Claude invokes a subagent it can pass a per-invocation `model`. Resolution order:

1. `CLAUDE_CODE_SUBAGENT_MODEL` env var
2. Per-invocation `model` parameter
3. Definition's `model` frontmatter
4. Main conversation's model

Values are checked against the organization's `availableModels` allowlist; an excluded resolution falls back to the inherited model.

## Invocation

Three patterns escalate from suggestion to session-wide default:

- **Natural language**: name the subagent; Claude decides whether to delegate ("Use the test-runner subagent to fix failing tests").
- **@-mention**: `@"code-reviewer (agent)"` or `@agent-<name>` guarantees that subagent runs for one task. Claude still writes the task prompt; the mention only controls which subagent.
- **Session-wide `--agent <name>`**: the main thread takes on the subagent's system prompt (replacing the default), tool restrictions, and model. CLAUDE.md still loads. Persists across resume. Can also be set via the `agent` key in `.claude/settings.json` (the CLI flag overrides it).

Automatic delegation is driven by the request, the `description` field, and current context; "use proactively" in the description encourages it.

**Foreground vs background**: foreground subagents block and pass prompts through; background subagents run concurrently and (as of v2.1.186) surface permission prompts in the main session naming the asker. `CLAUDE_CODE_FORK_SUBAGENT=1` forces every spawn to background.

**Nested subagents** (v2.1.172+): a subagent can spawn its own subagents up to a fixed depth of five; only the top-level summary returns. A fork can't spawn another fork but can spawn other types.

## Subagent hooks

Two configuration paths:

- **Frontmatter hooks** run only while that subagent is active and are cleaned up when it finishes. All hook events are supported; the common ones are `PreToolUse`, `PostToolUse`, and `Stop` (converted to `SubagentStop` at runtime).
- **Project-level hooks in `settings.json`** respond to subagent lifecycle in the main session:

| Event | Matcher input | When it fires |
| :---- | :------------ | :------------ |
| `SubagentStart` | Agent type name | When a subagent begins execution |
| `SubagentStop` | Agent type name | When a subagent completes |

Frontmatter hooks fire both when spawned as a subagent and when the agent runs as the main session via `--agent`.

## Forking the current conversation

> "A fork is a subagent that inherits the entire conversation so far instead of starting fresh."

A fork sees the same system prompt, tools, model, and message history as the main session — dropping the input isolation other subagents provide — while its own tool calls stay out of the conversation and only its final result returns. Requires v2.1.117+; `/fork` is enabled by default from v2.1.161, otherwise gated by `CLAUDE_CODE_FORK_SUBAGENT=1`. Because its system prompt and tools match the parent, its first request reuses the parent's prompt cache, making it cheaper than a fresh subagent.

| | Fork | Named subagent |
| :-- | :--- | :------------- |
| Context | Full conversation history | Fresh context + passed prompt |
| System prompt and tools | Same as main session | From the definition file |
| Model | Same as main session | From the `model` field |
| Prompt cache | Shared with main session | Separate cache |

A fork can't spawn another fork.

## Orchestration layer — who owns the harness

As subagent fleets scale, a human-role question emerges that the reference doesn't cover:

- [The agent manager role](../concepts/agent-manager-role.md) — who owns the harness (the agent definitions, hooks, and permission policy) at team scale, and how that ownership keeps a multi-subagent setup coherent. Pairs with [large-codebase strategies](large-codebase.md) for the broader scaling playbook.

## Relationship to concept pages

This summary is the primary source for:
- [Subagents](../concepts/subagents.md) — what they are, built-in types, when to use them
- [Subagent configuration](../concepts/subagent-configuration.md) — frontmatter fields, scopes, file format
- [Subagent tool access](../concepts/subagent-tool-access.md) — `tools`/`disallowedTools`, MCP scoping, permission modes
- [Subagent context](../concepts/subagent-context.md) — separate windows, startup loading, resume, compaction
- [Subagent invocation](../concepts/subagent-invocation.md) — delegation, @-mention, `--agent`, foreground/background, nesting
- [Subagent hooks](../concepts/subagent-hooks.md) — frontmatter hooks and `SubagentStart`/`SubagentStop`
- [Forked subagents](../concepts/forked-subagents.md) — inheriting forks vs named subagents
