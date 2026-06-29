---
type: concept
title: "Built-in Tools"
created: 2026-06-29
updated: 2026-06-29
tags: [tools, permissions, mcp, skills, reference, catalog]
source_count: 1
sources:
  - sources/clean/code-claude-com-docs-en-tools-reference-md.md
---

# Built-in Tools

> Claude Code has access to a set of built-in tools that help it understand and modify your codebase.

The **tool names are the exact strings** you reference elsewhere in the system — in [permission rules](./tool-permission-rules.md), [subagent tool lists](./subagent-tool-access.md), and [hook matchers](./hook-matchers.md). This page is the catalog: which tools exist, which require permission, and how the set is extended or narrowed.

## Naming, disabling, and extending

- **Disable a tool entirely** by adding its name to the `deny` array in your [permission settings](./permission-settings.md).
- **Add custom tools** by connecting an [MCP server](./plugin-components.md) — MCP tools appear alongside the built-ins.
- **Add reusable prompt-based workflows** by writing a [skill](./skills.md). A skill "runs through the existing `Skill` tool rather than adding a new tool entry" — it does not create a new tool name.

Your exact tool set depends on provider, platform, and settings. To see what is loaded in a running session, ask Claude `What tools do you have access to?` (it gives a conversational summary); for exact MCP tool names, run `/mcp`.

## The catalog

The **Permission** column marks tools whose calls are checked against your permission rules (`Yes`) versus tools that run without a permission gate (`No`). Disabling a `No`-permission tool still requires a `deny` rule.

### File and code

| Tool | Behavior | Permission |
| :--- | :------- | :--------- |
| `Read` | Reads file contents with line numbers; also renders images, PDFs, and notebooks. See [file tool behavior](./file-tool-behavior.md) | No |
| `Write` | Creates or overwrites a file with full content (no append/merge). See [file tool behavior](./file-tool-behavior.md) | Yes |
| `Edit` | Exact-string replacement on a file. See [file tool behavior](./file-tool-behavior.md) | Yes |
| `NotebookEdit` | Modifies one Jupyter cell at a time by `cell_id` | Yes |
| `Glob` | Finds files by name pattern (`**` recursive) | No |
| `Grep` | Searches file contents (ripgrep-backed) | No |
| `LSP` | Language-server code intelligence: definitions, references, type errors | No |

### Shell execution

| Tool | Behavior | Permission |
| :--- | :------- | :--------- |
| `Bash` | Runs shell commands. See [Bash execution model](./bash-execution-model.md) | Yes |
| `PowerShell` | Runs PowerShell commands natively (opt-in outside Windows) | Yes |
| `Monitor` | Runs a background command and feeds each output line back to Claude. Shares Bash permission rules. See [monitors vs command hooks](../comparisons/monitors-vs-command-hooks.md) | Yes |

### Web

| Tool | Behavior | Permission |
| :--- | :------- | :--------- |
| `WebFetch` | Fetches a URL and extracts an answer with a small model — **lossy by design** (the extraction prompt decides what reaches Claude) | Yes |
| `WebSearch` | Runs a query and returns titles and URLs only; it does not fetch pages — follow up with `WebFetch` | Yes |

`WebSearch` is the one tool whose rule takes **no specifier** — a bare `WebSearch` in `allow`/`deny` is the only form.

### Agents and orchestration

| Tool | Behavior | Permission |
| :--- | :------- | :--------- |
| `Agent` | Spawns a [subagent](./subagents.md) with its own context window; also launches [forked subagents](./forked-subagents.md) when fork mode is on | No |
| `SendMessage` | Messages an [agent-team](./subagent-invocation.md) teammate or resumes a subagent by agent ID | No |
| `Workflow` | Runs a dynamic workflow script that orchestrates many subagents in the background | Yes |
| `TaskCreate` / `TaskGet` / `TaskList` / `TaskUpdate` / `TaskStop` | Manage the task list and background tasks | No |
| `TaskOutput` | *(Deprecated)* — prefer `Read` on the task's output file | No |
| `TodoWrite` | Session checklist; disabled by default since v2.1.142 in favor of the `Task*` tools | No |

### Planning and interaction

| Tool | Behavior | Permission |
| :--- | :------- | :--------- |
| `EnterPlanMode` | Switches to plan mode to design before coding | No |
| `ExitPlanMode` | Presents a plan for approval and exits plan mode | Yes |
| `AskUserQuestion` | Asks multiple-choice questions to clarify ambiguity | No |

### Scheduling and notification

| Tool | Behavior | Permission |
| :--- | :------- | :--------- |
| `CronCreate` / `CronDelete` / `CronList` | Schedule/cancel/list session-scoped recurring or one-shot prompts | No |
| `ScheduleWakeup` | Reschedules the next iteration of a self-paced `/loop` | No |
| `RemoteTrigger` | Creates/runs Routines on claude.ai; backs `/schedule` | No |
| `PushNotification` | Sends a desktop notification (and a phone push with Remote Control) | No |

### Worktrees

| Tool | Behavior | Permission |
| :--- | :------- | :--------- |
| `EnterWorktree` | Creates an isolated git worktree and switches into it | No |
| `ExitWorktree` | Exits a worktree session and returns to the original directory | No |

### Skills, MCP, and tool discovery

| Tool | Behavior | Permission |
| :--- | :------- | :--------- |
| `Skill` | Executes a [skill](./skill-invocation-control.md) within the main conversation | Yes |
| `ToolSearch` | Searches for and loads deferred tools when [tool search](./plugin-components.md) is enabled | No |
| `WaitForMcpServers` | Waits for MCP servers still connecting (only when tool search is disabled) | No |
| `ListMcpResourcesTool` / `ReadMcpResourceTool` | List/read resources from connected MCP servers | No |

### Publishing and sharing

| Tool | Behavior | Permission |
| :--- | :------- | :--------- |
| `Artifact` | Publishes an HTML/Markdown file as a private interactive page on claude.ai (Team/Enterprise plan) | Yes |
| `ShareOnboardingGuide` | Uploads `ONBOARDING.md` and returns a share link | Yes |

## Provider availability

Several tools route through Anthropic-hosted infrastructure and are therefore **not available on Amazon Bedrock, Google Vertex AI, or Microsoft Foundry**: `PushNotification`, `RemoteTrigger`, `ScheduleWakeup`, and `Monitor`. `WebSearch` availability also varies by provider (works on the Claude API and Microsoft Foundry, and with Claude 4 models on Vertex AI; Bedrock does not expose it).

> The [advisor tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/advisor-tool) is a **server tool** the API runs, not a tool Claude Code implements — it has no name you can reference in permission rules or hook matchers.

## Related concepts

- [Tool Permission Rules](./tool-permission-rules.md) — the `ToolName(specifier)` format and where it is referenced
- [File Tool Behavior](./file-tool-behavior.md) — Read/Write/Edit/NotebookEdit/Glob/Grep specifics
- [Bash Execution Model](./bash-execution-model.md) — Bash persistence, limits, background tasks, PowerShell
- [Subagent Tool Access](./subagent-tool-access.md) — how a subagent's `tools` list narrows this catalog
- [Permission Settings](./permission-settings.md) — the `allow`/`ask`/`deny` blocks that gate these tools
