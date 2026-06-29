---
type: source
source_url: https://docs.claudekit.cc/docs/engineer/configuration/hooks/
title: "Hooks - ClaudeKit Documentation"
raw_capture: ../raw/docs-claudekit-cc-docs-engineer-configuration-hooks.html
captured: 2026-06-29
---

Search... ⌘K

Engineer Kit

Overview (1)

Engineer Kit

Agents (14)

Agents Overview

B

Brainstormer Agent

C

Code Reviewer Agent Code Simplifier Agent

D

Debugger Agent Docs Manager Agent

F

Fullstack Developer Agent

G

Git Manager Agent

J

Journal Writer Agent

P

Planner Agent Project Manager Agent

R

Researcher Agent

T

Tester Agent

U

UI/UX Designer Agent

Skills (83)

Skills Overview

A

agent-browser agentize ai-artist ai-multimodal ask autoresearch

B

backend-development better-auth bootstrap brainstorm

C

chrome-devtools code-review coding-level context-engineering cook copywriting cti-expert

D

databases debug deploy design devops docs docs-seeker document-skills

E

excalidraw

F

find-skills fix frontend-design frontend-development

G

git gkg google-adk-python graphify

J

journal

K

kanban

L

llms loop

M

markdown-novel-viewer mcp-builder mcp-management media-processing mermaidjs-v11 mintlify mobile-development

P

payment-integration plan plans-kanban predict preview problem-solving project-management project-organization

R

react-best-practices remotion repomix research retro

S

scenario scout security security-scan sequential-thinking shader ship shopify show-off skill-creator stitch

T

tanstack team test threejs

U

ui-styling ui-ux-pro-max use-mcp

W

watzup web-design-guidelines web-frameworks web-testing worktree

X

xia

Configuration (4)

CLAUDE.md Workflows Hooks MCP Setup

LLMs.txt

Getting Started Docs CLI Workflows Tools Support

English

Copy for AI

# Hooks

Hooks allow you to extend Claude Code with custom scripts that run at specific points in the workflow. ClaudeKit Engineer includes pre-built hooks for file access control, naming guidance, simplification gates, statusline rendering, optional session context, and notifications.

## Overview

Hooks are configured in .claude/settings.json and execute shell commands in response to Claude Code events.

Current default (v2.19.0+): ClaudeKit installs only lightweight safety and workflow hooks by default. Generated session/subagent/usage context hooks are opt-in and are pruned from existing installs during ck update or ck migrate.

### Available Hook Events

EventWhen Triggered

SessionStartWhen a Claude Code session begins

SubagentStartWhen a Task tool spawns a subagent

UserPromptSubmitBefore user prompt is processed

PreToolUseBefore a tool executes

PostToolUseAfter a tool executes

TaskCompletedWhen a task is marked completed

TeammateIdleWhen an Agent Team member goes idle

StopWhen Claude session ends

SubagentStopWhen a subagent completes

## Configuration

Hooks are defined in .claude/settings.json:

```
{
  "statusLine": {
    "type": "command",
    "command": "bash .claude/hooks/node-hook-runner.sh .claude/statusline.cjs",
    "padding": 0
  },
  "hooks": {
    "UserPromptSubmit": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "bash .claude/hooks/node-hook-runner.sh .claude/hooks/simplify-gate.cjs"
          }
        ]
      }
    ],
    "PreToolUse": [
      {
        "matcher": "Write",
        "hooks": [
          {
            "type": "command",
            "command": "bash .claude/hooks/node-hook-runner.sh .claude/hooks/descriptive-name.cjs"
          }
        ]
      },
      {
        "matcher": "Bash|Glob|Grep|Read|Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "bash .claude/hooks/node-hook-runner.sh .claude/hooks/scout-block.cjs"
          },
          {
            "type": "command",
            "command": "bash .claude/hooks/node-hook-runner.sh .claude/hooks/privacy-block.cjs"
          }
        ]
      }
    ]
  }
}
```

### Hook Properties

- type: Always "command" for shell execution

- command: Shell command to run

- matcher: (PreToolUse only) Regex to match tool names

## Built-in Hooks

ClaudeKit Engineer ships with hooks organized by event type. All default hook files live in .claude/hooks/. Shared utilities are in .claude/hooks/lib/. Notification providers are in .claude/hooks/notifications/.

### Summary Table

Hook FileDefaultEventPurpose

simplify-gate.cjsYesUserPromptSubmitEnforce simplification workflow rules when relevant

descriptive-name.cjsYesPreToolUse (Write)Inject file naming guidance: kebab-case, language conventions

scout-block.cjsYesPreToolUse (Bash/Glob/Grep/Read/Edit/Write)Block access to .ckignore-listed directories

privacy-block.cjsYesPreToolUse (Bash/Glob/Grep/Read/Edit/Write)Block sensitive files, require user approval

session-init.cjsNoSessionStartLoad config, detect project, persist env vars

subagent-init.cjsNoSubagentStartInject minimal context to subagents

team-context-inject.cjsNoSubagentStartInject peer info + task summary for Agent Team members

cook-after-plan-reminder.cjsNoSubagentStop (Plan)Print user-choice guidance after planning

dev-rules-reminder.cjsNoUserPromptSubmitInject session context, rules, modularization, Plan Context

usage-context-awareness.cjsNoUserPromptSubmit + PostToolUseFetch usage limits, write to cache

post-edit-simplify-reminder.cjsNoPostToolUse (Edit/Write/MultiEdit)Remind about code-simplifier after repeated edits

task-completed-handler.cjsNoTaskCompletedLog completions, inject progress for Agent Team leads

teammate-idle-handler.cjsNoTeammateIdleInject available task context when team member goes idle

The hooks marked No are no longer installed by default because they generate context on session, prompt, subagent, or tool events. Existing installations are cleaned up idempotently by the CLI during update and migration.

### session-init.cjs

Event: SessionStart

Purpose: Runs once when a Claude Code session begins. Loads .claude/.env, detects the current project, and persists environment variables for the session.

What it does:

- Reads .claude/.env and .env files and exports variables to the session

- Detects project type (Next.js, Express, etc.) and sets CK_PROJECT_TYPE

- Stores session metadata for hooks that run later in the session

### subagent-init.cjs

Event: SubagentStart

Purpose: Injects minimal context (~200 tokens) into subagents spawned by the Task tool. Keeps subagent prompts lean while ensuring they have the critical paths and config they need.

What it does:

- Injects CWD, plan directory, reports directory

- Passes CK_SESSION_ID and project type to subagents

- Does NOT inject full rules (those come from dev-rules-reminder.cjs)

### team-context-inject.cjs

Event: SubagentStart

Purpose: Used in Agent Team workflows. Injects peer agent names, task summary, and coordination context into team member subagents.

What it does:

- Reads team config from .claude/teams/

- Injects available peer names and their current task status

- Provides file ownership boundaries from the active task

Only active when CK_TEAM_MODE=1 is set.

### cook-after-plan-reminder.cjs

Event: SubagentStop (matcher: Plan subagents)

Purpose: After a planning subagent completes, prints boundary guidance so Claude stops before implementation and presents the available next steps.

What it does:

- Detects if the stopping subagent was a planner

- Prints an optional /ck:cook <plan.md> command with the generated plan path

- Keeps planning and implementation separated until the user approves implementation

- Mentions --auto only as an explicit opt-in for autonomous implementation

### dev-rules-reminder.cjs

Event: UserPromptSubmit

Purpose: Injects session context, development rules, modularization guidelines, and Plan Context into Claude’s context before each user prompt.

What it does:

- Reads .claude/workflows/development-rules.md and injects key rules

- Injects active Plan Context (current plan directory) from session state

- Reminds about modularization limits (200-line files)

- Ensures consistent code quality standards across prompts

### usage-context-awareness.cjs

Event: UserPromptSubmit + PostToolUse

Purpose: Fetches Claude Code usage limits from the API and writes them to a cache file. Throttled to avoid excessive API calls. Helps Claude be aware of remaining token budget.

What it does:

- Reads cached usage data from .claude/.usage-cache.json (max 60s old)

- Fetches fresh usage stats if cache is stale

- Injects remaining token budget into context

- Silently skips if ANTHROPIC_API_KEY is not set

### descriptive-name.cjs

Event: PreToolUse (matcher: Write)

Purpose: Injects file naming guidance before Claude writes a new file, enforcing kebab-case and language-specific conventions.

What it does:

- Detects the target file path from the Write tool parameters

- Checks if the name follows kebab-case (for most files) or language conventions (e.g., PascalCase for React components)

- Injects a reminder if the name is non-descriptive or uses the wrong convention

### scout-block.cjs

Event: PreToolUse (matcher: Bash|Glob|Grep|Read|Edit|Write)

Purpose: Blocks file system access to directories listed in .ckignore. Allows build commands (e.g., npm run build) to pass through.

What it does:

- Reads the shipped .ckignore baseline and then layers an optional git-root ./.claude/.ckignore override

- Blocks Read/Edit/Write/Glob/Grep operations on matched paths

- Allows Bash commands that are recognized build/test commands

- Returns a clear error message explaining which path is blocked and which .ckignore file to edit

### privacy-block.cjs

Event: PreToolUse (matcher: Bash|Glob|Grep|Read|Edit|Write)

Purpose: Blocks access to sensitive files (.env, credentials, private keys). Requires explicit user approval via AskUserQuestion before allowing access.

What it does:

- Matches file paths against a list of sensitive patterns (.env*, *.pem, *.key, credentials.*, etc.)

- On match: returns a @@PRIVACY_PROMPT@@ JSON marker that triggers the AskUserQuestion flow

- If user approves: Claude uses bash cat to read the file (bypasses the hook)

- If user denies: continues without the sensitive file

See $HOME/.claude/CLAUDE.md → “Hook Response Protocol” for the full @@PRIVACY_PROMPT@@ flow.

### post-edit-simplify-reminder.cjs

Event: PostToolUse (matcher: Edit|Write|MultiEdit)

Purpose: Tracks the number of file edits in the current session. After 5 or more edits, injects a reminder to run the code-simplifier agent to review accumulated complexity.

What it does:

- Increments edit counter in session state after each Edit/Write/MultiEdit

- At threshold (default: 5), injects: “You’ve made N edits. Consider invoking /simplify to review accumulated complexity.”

- Resets counter after reminder is shown

### task-completed-handler.cjs

Event: TaskCompleted

Purpose: Logs task completions to .claude/logs/tasks.log. In Agent Team mode, injects progress summary for the lead agent.

What it does:

- Appends task ID, subject, completion time, and owner to the task log

- If CK_TEAM_MODE=1: formats a progress summary showing completed vs total tasks and injects it into lead’s context

### teammate-idle-handler.cjs

Event: TeammateIdle

Purpose: When an Agent Team member goes idle (waiting for input), injects available unblocked task context so they can claim the next task without waiting for a message.

What it does:

- Reads TaskList for unblocked pending tasks

- Formats task summary and injects it as context

- Prevents teammates from staying idle when work is available

### Discord Notifications

File: .claude/hooks/send-discord.sh

Purpose: Sends rich notifications to Discord when tasks complete.

Note: Discord notifications are triggered manually in workflows, not automatically via hook events. This is intentional for flexibility.

Setup:

-

Create Discord Webhook:

- Discord Server → Settings → Integrations → Webhooks

- Create webhook, copy URL

-

Configure Environment:

```
# .env or .claude/.env
DISCORD_WEBHOOK_URL=https://discord.com/api/webhooks/YOUR_ID/YOUR_TOKEN
```

-

Make Executable:

```
chmod +x .claude/hooks/send-discord.sh
```

-

Test:

```
./.claude/hooks/send-discord.sh 'Test notification'
```

Usage in Workflows:

```
<!-- In .claude/workflows/development-rules.md -->
- When implementation complete, run:
  `./.claude/hooks/send-discord.sh 'Task completed: [summary]'`
```

Message Format:

```
╔═══════════════════════════════════════╗
║ 🤖 Claude Code Session Complete       ║
╠═══════════════════════════════════════╣
║ Implementation Complete               ║
║                                       ║
║ ✅ Added user authentication          ║
║ ✅ Created login/signup forms         ║
║ ✅ All tests passing                  ║
╠═══════════════════════════════════════╣
║ ⏰ Session Time: 14:30:45             ║
║ 📂 Project: my-project                ║
╚═══════════════════════════════════════╝
```

### Telegram Notifications

File: .claude/hooks/telegram_notify.sh

Purpose: Sends detailed notifications to Telegram with tool usage stats.

Setup:

-

Create Telegram Bot:

- Message @BotFather on Telegram

- Send /newbot, follow prompts

- Copy bot token

-

Get Chat ID:

```
# After messaging your bot, run:
curl -s "https://api.telegram.org/bot<TOKEN>/getUpdates" | jq '.result[-1].message.chat.id'
```

-

Configure Environment:

```
# .env or .claude/.env
TELEGRAM_BOT_TOKEN=123456789:ABCdefGHIjklMNOpqrsTUVwxyz
TELEGRAM_CHAT_ID=987654321
```

-

Configure Hook (add to .claude/settings.json):

Note: Telegram hooks are not configured by default. Add this to your settings.json to enable automatic notifications.

```
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/telegram_notify.sh"
          }
        ]
      }
    ],
    "SubagentStop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/telegram_notify.sh"
          }
        ]
      }
    ]
  }
}
```

Message Format:

```
🚀 Project Task Completed

📅 Time: 2025-10-22 14:30:45
📁 Project: my-project
🔧 Total Operations: 15
🆔 Session: abc12345...

Tools Used:
   5 Edit
   3 Read
   2 Bash
   2 Write

Files Modified:
• src/auth/service.ts
• src/utils/validation.ts
• tests/auth.test.ts

📍 Location: /Users/user/projects/my-project
```

## Notification System

Notification providers live in .claude/hooks/notifications/. Each provider is a standalone module that reads from environment variables.

ProviderFileEvent

Discordnotifications/discord.cjsStop, SubagentStop

Slacknotifications/slack.cjsStop, SubagentStop

Telegramnotifications/telegram.cjsStop, SubagentStop

Configure providers by setting the relevant env vars (see Discord/Telegram sections below) and adding them to settings.json hooks. The provider scripts are called by the main notification orchestrator.

## Creating Custom Hooks

### Basic Hook Structure

```
// .claude/hooks/my-hook.cjs

// Read hook input from stdin (JSON)
let input = '';
process.stdin.on('data', chunk => input += chunk);
process.stdin.on('end', () => {
  const data = JSON.parse(input);

  // Hook logic here
  console.log('Hook triggered:', data.hookType);

  // Exit 0 for success, non-zero to block
  process.exit(0);
});
```

### Hook Input Data

UserPromptSubmit:

```
{
  "hookType": "UserPromptSubmit",
  "projectDir": "/path/to/project",
  "prompt": "User's prompt text"
}
```

PreToolUse:

```
{
  "hookType": "PreToolUse",
  "projectDir": "/path/to/project",
  "tool": "Edit",
  "parameters": {
    "file_path": "/path/to/file.ts",
    "old_string": "...",
    "new_string": "..."
  }
}
```

Stop:

```
{
  "hookType": "Stop",
  "projectDir": "/path/to/project",
  "sessionId": "abc123",
  "toolsUsed": [
    {"tool": "Read", "parameters": {"file_path": "..."}},
    {"tool": "Edit", "parameters": {"file_path": "..."}}
  ]
}
```

### Example: Logging Hook

```
// .claude/hooks/log-tools.cjs
const fs = require('fs');

let input = '';
process.stdin.on('data', chunk => input += chunk);
process.stdin.on('end', () => {
  const data = JSON.parse(input);

  const logEntry = {
    timestamp: new Date().toISOString(),
    hookType: data.hookType,
    tool: data.tool,
    file: data.parameters?.file_path
  };

  fs.appendFileSync('logs.txt', JSON.stringify(logEntry) + '\n');
  process.exit(0);
});
```

### Example: Blocking Hook

```
// .claude/hooks/prevent-secrets.cjs
let input = '';
process.stdin.on('data', chunk => input += chunk);
process.stdin.on('end', () => {
  const data = JSON.parse(input);

  // Block edits to .env files
  if (data.tool === 'Edit' && data.parameters?.file_path?.includes('.env')) {
    console.error('Blocked: Cannot edit .env files directly');
    process.exit(1); // Non-zero exits block the action
  }

  process.exit(0);
});
```

## Environment Variables

Hooks can access these environment variables:

VariableDescription

CLAUDE_PROJECT_DIRProject root directory

CLAUDE_SESSION_IDCurrent session identifier

Custom variablesFrom .env files

### Loading .env Files

ClaudeKit hooks load environment variables in this priority:

- System environment variables

- .claude/.env (project-level)

- .claude/hooks/.env (hook-specific)

## Best Practices

### Security

-

Never commit secrets:

```
# .gitignore
.env
.env.*
```

-

Use environment variables for tokens and URLs

-

Rotate webhook tokens regularly

-

Limit hook permissions to necessary scope

### Performance

- Keep hooks lightweight - they run on every event

- Use async operations for slow tasks

- Exit quickly if no action needed

### Reliability

- Handle errors gracefully

- Log hook failures for debugging

- Test hooks manually before deployment

## Troubleshooting

### Hook Not Triggering

Solutions:

- Verify hook in settings.json is valid JSON

- Check script is executable (chmod +x)

- Verify path is correct

- Test script manually

### Hook Blocking Unexpectedly

Solutions:

- Check exit code (0 = allow, non-zero = block)

- Review matcher regex for PreToolUse

- Add logging to debug

### Environment Variables Not Loading

Solutions:

- Check .env file exists and has correct format

- Verify no spaces around = in .env

- Ensure script reads .env files

## Related

- CLAUDE.md - Project instructions

- MCP Setup - MCP server configuration

- Workflows - Development workflows

Key Takeaway: Hooks extend Claude Code with custom automation - from development rule enforcement to real-time notifications. Use built-in Discord/Telegram hooks or create custom hooks to fit your workflow.

## On this page

- Overview

- Available Hook Events

- Configuration

- Hook Properties

- Built-in Hooks

- Summary Table

- session-init.cjs

- subagent-init.cjs

- team-context-inject.cjs

- cook-after-plan-reminder.cjs

- dev-rules-reminder.cjs

- usage-context-awareness.cjs

- descriptive-name.cjs

- scout-block.cjs

- privacy-block.cjs

- post-edit-simplify-reminder.cjs

- task-completed-handler.cjs

- teammate-idle-handler.cjs

- Discord Notifications

- Telegram Notifications

- Notification System

- Creating Custom Hooks

- Basic Hook Structure

- Hook Input Data

- Example: Logging Hook

- Example: Blocking Hook

- Environment Variables

- Loading .env Files

- Best Practices

- Security

- Performance

- Reliability

- Troubleshooting

- Hook Not Triggering

- Hook Blocking Unexpectedly

- Environment Variables Not Loading

- Related

## Search documentation

ESC

Type to search documentation...

↵ select ↑↓ navigate esc close
