---
type: summary
title: "Tools reference — built-in tools and their behavior"
slug: code-claude-com-docs-en-tools-reference-md
created: 2026-06-29
updated: 2026-06-29
tags: [tools, reference, permissions, bash, file-tools, mcp]
sources:
  - sources/clean/code-claude-com-docs-en-tools-reference-md.md
---

# Summary: Tools reference

The authoritative reference for the built-in tools Claude Code can use, including which tools require permission, the exact tool-name strings used in permission rules / subagent lists / hook matchers, and per-tool behavior notes. Tool names are exact strings; to disable a tool entirely, add its name to the `deny` array in permission settings.

## Key thesis from source

> "Complete reference for the tools Claude Code can use, including permission requirements and per-tool behavior."

Claude generally decides when to use these tools. You name them yourself only when configuring permissions, subagent tool lists, skill `allowed-tools`, or hook matchers. Custom tools come from connecting an [MCP server](../concepts/built-in-tools.md); reusable prompt workflows are skills, which run through the existing `Skill` tool rather than adding a new tool entry.

## Built-in tools

The full set, mirroring the source table. "Permission Required" means the tool prompts (or is gated by allow/deny rules) before acting.

| Tool | Description | Permission Required |
| :--- | :--- | :---: |
| `Agent` | Spawns a subagent with its own context window to handle a task | No |
| `Artifact` | Publishes an HTML/Markdown file as an artifact on claude.ai (Team/Enterprise plan, `/login`) | Yes |
| `AskUserQuestion` | Asks multiple-choice questions to gather requirements or clarify ambiguity | No |
| `Bash` | Executes shell commands in your environment | Yes |
| `CronCreate` | Schedules a recurring or one-shot prompt within the session (session-scoped) | No |
| `CronDelete` | Cancels a scheduled task by ID | No |
| `CronList` | Lists all scheduled tasks in the session | No |
| `Edit` | Makes targeted edits to specific files | Yes |
| `EnterPlanMode` | Switches to plan mode to design an approach before coding | No |
| `EnterWorktree` | Creates an isolated git worktree and switches into it | No |
| `ExitPlanMode` | Presents a plan for approval and exits plan mode | Yes |
| `ExitWorktree` | Exits a worktree session and returns to the original directory | No |
| `Glob` | Finds files based on pattern matching | No |
| `Grep` | Searches for patterns in file contents | No |
| `ListMcpResourcesTool` | Lists resources exposed by connected MCP servers | No |
| `LSP` | Code intelligence via language servers (definitions, references, type errors) | No |
| `Monitor` | Runs a command in the background and feeds each output line back to Claude | Yes |
| `NotebookEdit` | Modifies Jupyter notebook cells | Yes |
| `PowerShell` | Executes PowerShell commands natively | Yes |
| `PushNotification` | Sends a desktop notification (and phone push with Remote Control) | No |
| `Read` | Reads the contents of files | No |
| `ReadMcpResourceTool` | Reads a specific MCP resource by URI | No |
| `RemoteTrigger` | Creates, updates, runs, lists Routines on claude.ai (backs `/schedule`) | No |
| `ScheduleWakeup` | Reschedules the next iteration of a self-paced `/loop` | No |
| `SendMessage` | Messages an agent-team teammate, or resumes a subagent by agent ID | No |
| `ShareOnboardingGuide` | Uploads `ONBOARDING.md` and returns a share link | Yes |
| `Skill` | Executes a skill within the main conversation | Yes |
| `TaskCreate` | Creates a new task in the task list | No |
| `TaskGet` | Retrieves full details for a specific task | No |
| `TaskList` | Lists all tasks with their current status | No |
| `TaskOutput` | (Deprecated) Retrieves output from a background task; prefer `Read` on the output file | No |
| `TaskStop` | Kills a running background task by ID | No |
| `TaskUpdate` | Updates task status, dependencies, details, or deletes tasks | No |
| `TodoWrite` | Manages the session task checklist (disabled by default as of v2.1.142) | No |
| `ToolSearch` | Searches for and loads deferred tools when tool search is enabled | No |
| `WaitForMcpServers` | Waits for MCP servers still connecting in the background | No |
| `WebFetch` | Fetches content from a specified URL | Yes |
| `WebSearch` | Performs web searches | Yes |
| `Workflow` | Runs a dynamic workflow orchestrating many subagents in the background | Yes |
| `Write` | Creates or overwrites files | Yes |

## Permission rule formats

Tool names plug into `ToolName(specifier)` rules. Several tools share a specifier format; an `Edit(...)` allow rule also grants read access to the same path.

| Rule format | Applies to | Match style |
| :--- | :--- | :--- |
| `Bash(npm run *)` | Bash, Monitor | Command pattern matching |
| `PowerShell(Get-ChildItem *)` | PowerShell | Command pattern matching |
| `Read(~/secrets/**)` | Read, Grep, Glob, LSP | Path pattern matching |
| `Edit(/src/**)` | Edit, Write, NotebookEdit | Path pattern matching |
| `Skill(deploy *)` | Skill | Skill name matching |
| `Agent(Explore)` | Agent | Subagent type matching |
| `WebFetch(domain:example.com)` | WebFetch | Domain matching |
| `WebSearch` | WebSearch | No specifier; allow/deny whole tool |

Tools not listed (e.g. `ExitPlanMode`, `ShareOnboardingGuide`) accept only the bare tool name. Hook `matcher` fields use bare tool names, not the parenthesized rule format.

## Bash execution model

See [bash execution model](../concepts/bash-execution-model.md). The Bash tool runs each command in a separate process:

- **Working directory carry-over**: `cd` in the main session persists to later Bash commands, but only inside the project directory or an added `--add-dir` directory. A `cd` outside resets to the project directory and appends `Shell cwd was reset to <dir>`. Subagent sessions never carry over `cd`. Disable carry-over with `CLAUDE_BASH_MAINTAIN_PROJECT_WORKING_DIR=1`.
- **Env vars don't persist**: an `export` in one command is gone by the next. Use `CLAUDE_ENV_FILE` or a SessionStart hook to persist.
- **Aliases/functions persist**: Claude Code sources `~/.zshrc`/`~/.bashrc`/`~/.profile` at session start and applies them to every command.
- **Timeout**: 2 minutes default, up to 10 minutes via the `timeout` parameter (`BASH_DEFAULT_TIMEOUT_MS` / `BASH_MAX_TIMEOUT_MS`).
- **Output length**: 30,000 chars default; longer output is saved to a session file and Claude gets the path plus a preview. Hard ceiling 150,000 (`BASH_MAX_OUTPUT_LENGTH`).
- **Background**: `run_in_background: true` starts a long-running task; manage with `/tasks`.

PowerShell shares the same working-directory reset behavior and `CLAUDE_BASH_MAINTAIN_PROJECT_WORKING_DIR` variable.

## File tool semantics

See [file tool behavior](../concepts/file-tool-behavior.md).

**Read** returns contents with line numbers; Claude always passes absolute paths. A whole-file read over the token limit returns a first page with a `PARTIAL view` notice and `offset`/`limit` guidance; an explicit `offset`/`limit` read that still overflows errors. Read handles images (as visual content, possibly downscaled), PDFs (whole if short; `pages` ranges up to 20 at a time over 10 pages), and `.ipynb` notebooks (all cells with outputs). Read does not read directories — use `ls` via Bash.

**Edit** performs exact string replacement (no regex, no fuzzy). Three checks must pass:

| Check | Requirement |
| :--- | :--- |
| Read-before-edit | File must have been read this conversation and unchanged on disk since |
| Match | `old_string` must appear exactly, whitespace included |
| Uniqueness | `old_string` must appear once, or use longer context / `replace_all: true` |

> "Viewing a file with Bash also satisfies the read-before-edit requirement when the command is `cat`, `head`, `tail`, `sed -n 'X,Yp'`, `grep`, `egrep`, or `fgrep` on a single file with no pipes or redirects."

Read/Edit deny rules also apply to recognized file commands in Bash (`cat`, `head`, `tail`, `sed`, `grep`) but not to subprocesses that open files indirectly (e.g. a Python script); the read-before-edit list and the deny-rule list differ (`egrep`/`fgrep` count for the former, not the latter).

**Write** creates or overwrites a file with the full content (no append/merge). Overwriting an existing path requires that the file was read at least once this conversation (Bash viewing also satisfies this); new files are exempt. For partial changes, Claude uses Edit.

**NotebookEdit** edits one cell at a time by `cell_id` with `replace` (default), `insert`, or `delete` modes; permission rules use the `Edit(...)` path format.

## Search and code-intelligence tools

- **Glob** matches by name pattern (`**` recursive), sorts by modification time, caps at 100 files, and does *not* respect `.gitignore` by default (`CLAUDE_CODE_GLOB_NO_IGNORE=false` to enable).
- **Grep** is built on ripgrep (ripgrep regex, not POSIX); output modes `files_with_matches` (default), `content`, `count`; scope by `glob`/`type`; `multiline: true` for cross-line. Grep *does* respect `.gitignore`.
- **LSP** gives language-server intelligence, auto-reporting type errors after edits; inactive until a code-intelligence plugin is installed.

## Web tools

- **WebFetch** fetches a URL, converts HTML to Markdown, and runs the supplied prompt against the content via a small fast model — so it is **lossy by design** (Claude usually sees the model's answer, not the raw page). HTTP upgrades to HTTPS; large pages truncate; responses cache 15 minutes; cross-host redirects return a notice rather than following. Prompts on first reach of a new domain except preapproved doc domains; `WebFetch(domain:...)` rules override the preapproved set.
- **WebSearch** queries Anthropic's web-search backend, returning titles and URLs only (no page fetch — follow up with WebFetch). Up to eight backend searches per call; scope with `allowed_domains` or `blocked_domains` (not both). Permission rules take no specifier.

## Agent and Monitor notes

The **Agent** tool spawns a subagent in a separate context window that returns only a single text result; the parent never sees intermediate tool calls. Launching does not itself prompt — the subagent's own tool calls are checked against permission rules as it runs (foreground prompts inline; background subagents surface prompts in the main session as of v2.1.186). Subagent tool access follows `tools` / `disallowedTools` frontmatter, where `disallowedTools` takes precedence when both are set.

**Monitor** runs a background watch and feeds each output line back to Claude mid-conversation. It uses the same permission rules as Bash and is unavailable on Bedrock/Vertex/Foundry or when telemetry is disabled.

## Checking the available tool set

The exact tool set depends on provider, platform, and settings. Ask Claude "What tools do you have access to?" for a summary, or run `/mcp` for exact MCP tool names. The advisor tool is a server tool the API runs, with no referenceable name.

## Relationship to concept pages

This reference is the primary source for:
- [Built-in tools](../concepts/built-in-tools.md) — the tool catalog, permission requirements, and rule formats
- [File tool behavior](../concepts/file-tool-behavior.md) — Read/Write/Edit/NotebookEdit semantics and read-before-edit
- [Bash execution model](../concepts/bash-execution-model.md) — per-command process model, cwd carry-over, timeouts, output limits
