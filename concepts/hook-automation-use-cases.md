---
type: concept
title: "Hook Automation Use Cases"
created: 2026-06-29
updated: 2026-06-29
tags: [hooks, automation, use-cases, recipes, workflow]
source_count: 1
sources:
  - sources/clean/mlops-community-blog-the-complete-guide-to-claude-code-hooks-automating-your-ai-coding-workflow.md
---

# Hook Automation Use Cases

The [hook lifecycle reference](./hook-lifecycle-events.md) catalogs *which* events fire and *when*. This page is the practitioner companion: *what you can actually do* at each lifecycle point. It collects concrete automation recipes, grouped by the cadence at which the relevant hooks fire.

> "Hooks are the difference between Claude Code as a clever autocomplete and Claude Code as a first-class citizen in your engineering workflow. Start small. A lint-on-write PostToolUse hook and a rm -rf blocker in PreToolUse will already change how much you trust Claude to work autonomously."

## Session lifecycle (`SessionStart`, `SessionEnd`, `InstructionsLoaded`)

Prime context, persist environment, and clean up:

- Auto-load the team's current sprint tickets into Claude's context on `SessionStart`
- Set up project-specific environment variables (Node version, API keys, feature flags) so every Bash command Claude runs has them — persisted via `CLAUDE_ENV_FILE`
- Log session metadata to a metrics dashboard for auditing how the team uses Claude
- Clean up temporary files or close database connections on `SessionEnd` (no decision control here — cleanup only)

## User prompt (`UserPromptSubmit`)

Intercept, enrich, or block what the user is asking before it reaches the model:

- Block prompts that contain production credentials or PII before they ever reach the model
- Automatically append repo-specific context (current branch, recent commits, open PRs) to every prompt via `additionalContext`
- Auto-generate session titles using `sessionTitle` based on the first prompt
- Route certain prompts to a specialized agent by injecting instructions

## Tool execution (`PreToolUse`, `PostToolUse`, `PostToolUseFailure`)

> "This is where most of the magic happens."

- Block destructive Bash commands (`rm -rf`, `DROP TABLE`, force-push to main) before they run
- Auto-format and lint any file Claude edits via `PostToolUse` on `Edit`/`Write`
- Automatically run tests after every code change and feed results back to Claude
- Rewrite `npm`/`pip install` commands to use the company mirror via `updatedInput`
- Log every MCP tool call for audit trails in regulated environments
- Detect common tool failures (auth expired, rate limited) on `PostToolUseFailure` and tell Claude how to recover

## Permission (`PermissionRequest`, `PermissionDenied`)

Approve or deny programmatically instead of showing a dialog:

- Auto-approve Bash commands that match a whitelist (read-only git ops, test commands, linters) without prompting
- Auto-deny writes to sensitive paths (`.env`, `secrets/`, `production/`)
- Add "always allow" rules dynamically based on which team member is using Claude
- Tell Claude to retry a denied tool call with a different approach (`retry: true`)
- Modify tool input at permission time — e.g. force a dry-run flag

## Subagents & tasks (`SubagentStart`, `SubagentStop`, `TaskCreated`, `TaskCompleted`)

Observe and gate nested agentic activity:

- Enforce that every task subject starts with a ticket ID like `[PROJ-123]`
- Block task completion until the full test suite passes
- Inject security guidelines into every subagent that does code exploration
- Prevent `Explore` subagents from accessing certain directories
- Require a linked design doc before any task involving architecture changes can be created

## Stop & completion (`Stop`, `StopFailure`, `TeammateIdle`)

Decide whether Claude really gets to stop:

- Prevent Claude from stopping until all TODOs in the changed files are resolved
- Require passing lint and type checks before any `Stop` is allowed
- Auto-retry on transient API errors by detecting `rate_limit` in `StopFailure`
- Block teammate idle until a CI pipeline has been triggered

## Compaction (`PreCompact`, `PostCompact`)

- Block manual `/compact` unless the user passes specific instructions
- Archive the full conversation to external storage before compaction drops context
- Detect when context windows are filling too fast and warn the user

## File & environment (`CwdChanged`, `FileChanged`, `ConfigChange`)

direnv-style reactive configuration:

- Reload Node/Python versions automatically when Claude `cd`s into a different project
- Pick up new `.env` values the moment the file changes on disk
- Block unauthorized changes to `.claude/settings.json` outside of code review
- Watch `package.json` and reinstall dependencies when it changes

## Worktrees (`WorktreeCreate`, `WorktreeRemove`)

Swap the default git worktree behavior for custom VCS or setup:

- Use SVN, Mercurial, or Perforce instead of git for isolated sessions
- Copy `.env` and other gitignored config files into new worktrees automatically
- Archive worktree state before cleanup for debugging failed subagent runs

## MCP elicitation & notifications (`Elicitation`, `ElicitationResult`, `Notification`)

- Auto-fill MCP authentication prompts from your secret manager
- Desktop-notify the team chat whenever Claude needs a permission decision
- Route elicitation requests to a Slack channel for async team approval
- Send a phone alert when Claude has been idle too long on a long-running task

## Related concepts

- [Production Hook Patterns (ClaudeKit Case Study)](./production-hook-patterns.md) — how a real shipping kit applies these recipes, and the design lessons that emerge
- [Hook Lifecycle Events](./hook-lifecycle-events.md) — the full event reference these recipes attach to
- [Hook Types](./hook-types.md) — `command`/`http`/`prompt`/`agent` handlers and the `async` flag that make these recipes possible
- [Hook Decision Control](./hook-decision-control.md) — the `block`/`deny`/`allow`/`additionalContext` mechanics behind the blocking and enrichment recipes
- [Hook Scope](./hook-scope.md) — where to define these hooks (user, project, plugin, managed)
- [Subagent Hooks](./subagent-hooks.md) — gating nested subagent activity
- [Permission Evaluation](./permission-evaluation.md) — how auto-allow/deny recipes interact with the permission system
