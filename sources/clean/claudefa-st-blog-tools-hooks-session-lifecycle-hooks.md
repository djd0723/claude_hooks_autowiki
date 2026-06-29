---
type: source
source_url: https://claudefa.st/blog/tools/hooks/session-lifecycle-hooks
title: "Claude Code Session Hooks: Auto-Load Context Every Time"
raw_capture: ../raw/claudefa-st-blog-tools-hooks-session-lifecycle-hooks.html
captured: 2026-06-29
---

Claude Fast Code Kit 5.5 is here: Fable-optimized, with resumable nested sub-agents and dynamic workflows. Read the playbook.

Claude Fast

Claude Fast

Tools

Supercharge your Claude

Search

⌘K

Claude Code ToolsKeyboard ShortcutsStatus Line Guide

Hooks

Hooks GuideCross-Platform HooksSetup HooksStop HookSelf-Improving CLAUDE.mdSelf-Validating AgentsSession LifecycleContext Recovery HookSkill Activation HookPermission Hook

Skills

Orchestrators

Monitors

Customization

Resources

MCP & Extensions

Extensions

SEO Boost

Get Claude Fast

On this page

Hooks

# Claude Code Session Hooks: Auto-Load Context Every Time

SessionStart, SessionEnd, Setup, and PreCompact hooks for Claude Code. Auto-load context at startup and clean up on session end automatically.

Stop configuring. Start shipping.Everything you're reading about and more..
Agentic Orchestration Kit for Claude Code.

Get Claude Fast

Problem: Every time you start a Claude Code session, you manually remind it about your project state, environment setup, or current tasks. When sessions end, cleanup tasks are forgotten.

Quick Win: Add this SessionStart hook and Claude always knows your git state:

```
{
  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "echo '## Git' && git branch --show-current && git status --short | head -10"
          }
        ]
      }
    ]
  }
}
```

Now every session starts with context. Zero manual setup.

## The Session Lifecycle Hooks

Four hooks control session lifecycle:

HookWhen It FiresCan Block?Use Case

SetupWith --init or --maintenanceNOOne-time setup, migrations

SessionStartEvery session start/resumeNOLoad context, set env vars

PreCompactBefore context compactionNOBackup transcripts

SessionEndSession terminatesNOCleanup, logging

## SessionStart: Load Context Every Time

SessionStart fires when sessions begin or resume. Use it for context that should always be present.

### Basic Context Injection

```
{
  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "echo '## Project State' && cat .claude/tasks/session-current.md 2>/dev/null || echo 'No active session'"
          }
        ]
      }
    ]
  }
}
```

### With JSON Output

For structured context injection:

```
#!/usr/bin/env python3
import json
import sys
import subprocess

def get_project_context():
    try:
        branch = subprocess.check_output(
            ['git', 'rev-parse', '--abbrev-ref', 'HEAD'],
            text=True, stderr=subprocess.DEVNULL
        ).strip()
        status = subprocess.check_output(
            ['git', 'status', '--porcelain'],
            text=True, stderr=subprocess.DEVNULL
        ).strip()
        changes = len(status.split('\n')) if status else 0
    except:
        branch, changes = "unknown", 0

    return f"""=== SESSION CONTEXT ===
Git Branch: {branch}
Uncommitted Changes: {changes}
=== END ===""".strip()

output = {
    "hookSpecificOutput": {
        "hookEventName": "SessionStart",
        "additionalContext": get_project_context()
    }
}
print(json.dumps(output))
sys.exit(0)
```

### SessionStart Matchers

Target specific session events:

```
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup",
        "hooks": [{ "type": "command", "command": "echo 'Fresh session'" }]
      },
      {
        "matcher": "resume",
        "hooks": [{ "type": "command", "command": "echo 'Resumed session'" }]
      },
      {
        "matcher": "compact",
        "hooks": [{ "type": "command", "command": "echo 'Post-compaction'" }]
      }
    ]
  }
}
```

- startup - New session

- resume - From --resume, --continue, or /resume

- clear - After /clear

- compact - After compaction

### Persist Environment Variables

SessionStart has access to CLAUDE_ENV_FILE for setting session-wide environment variables:

```
#!/bin/bash

# Persist environment changes from nvm, pyenv, etc.
ENV_BEFORE=$(export -p | sort)

# Setup commands that modify environment
source ~/.nvm/nvm.sh
nvm use 20

if [ -n "$CLAUDE_ENV_FILE" ]; then
  ENV_AFTER=$(export -p | sort)
  comm -13 <(echo "$ENV_BEFORE") <(echo "$ENV_AFTER") >> "$CLAUDE_ENV_FILE"
fi

exit 0
```

Variables written to CLAUDE_ENV_FILE are available in all subsequent bash commands Claude runs.

## Setup: One-Time Operations

Setup hooks run only when explicitly invoked with --init, --init-only, or --maintenance. Use them for operations you don't want on every session.

### When to Use Setup vs SessionStart

OperationUse SetupUse SessionStart

Install dependenciesYesNo

Run database migrationsYesNo

Load git statusNoYes

Set environment variablesYesYes

Inject project contextNoYes

Cleanup temp filesYes (maintenance)No

### Setup Configuration

```
{
  "hooks": {
    "Setup": [
      {
        "matcher": "init",
        "hooks": [
          {
            "type": "command",
            "command": "npm install && npm run db:migrate"
          }
        ]
      },
      {
        "matcher": "maintenance",
        "hooks": [
          {
            "type": "command",
            "command": "npm prune && npm dedupe && rm -rf .cache"
          }
        ]
      }
    ]
  }
}
```

Invoke with:

```
claude --init          # Runs 'init' matcher
claude --init-only     # Runs 'init' matcher, then exits
claude --maintenance   # Runs 'maintenance' matcher
```

Setup hooks also have access to CLAUDE_ENV_FILE for persisting environment variables.

## PreCompact: Before Context Loss

PreCompact fires before compaction (manual /compact or automatic when context fills).

### Backup Transcripts

```
#!/usr/bin/env python3
import json
import sys
import shutil
from pathlib import Path
from datetime import datetime

input_data = json.load(sys.stdin)
transcript_path = input_data.get('transcript_path', '')
trigger = input_data.get('trigger', 'unknown')

if transcript_path and Path(transcript_path).exists():
    backup_dir = Path('.claude/backups')
    backup_dir.mkdir(parents=True, exist_ok=True)

    timestamp = datetime.now().strftime('%Y%m%d_%H%M%S')
    backup_name = f"transcript_{trigger}_{timestamp}.jsonl"
    shutil.copy2(transcript_path, backup_dir / backup_name)

    # Keep only last 10 backups
    backups = sorted(backup_dir.glob('transcript_*.jsonl'))
    for old_backup in backups[:-10]:
        old_backup.unlink()

sys.exit(0)
```

### PreCompact Matchers

```
{
  "hooks": {
    "PreCompact": [
      {
        "matcher": "auto",
        "hooks": [{ "type": "command", "command": "echo 'Auto-compacting...'" }]
      },
      {
        "matcher": "manual",
        "hooks": [{ "type": "command", "command": "echo 'Manual /compact'" }]
      }
    ]
  }
}
```

- auto - Context window filled, automatic compaction

- manual - User ran /compact

### Create Recovery Markers

Use PreCompact with SessionStart for context recovery. See the Context Recovery Hook for the complete pattern. The Code Kit includes a ContextRecoveryHook that implements this with a three-file architecture: a shared backup-core module, a statusline monitor for threshold-based triggers, and a PreCompact handler -- all coordinating through a shared state file so you never lose session context.

## SessionEnd: Cleanup

SessionEnd fires when sessions terminate. It cannot block termination but can perform cleanup.

### Log Session Stats

```
#!/usr/bin/env python3
import json
import sys
from pathlib import Path
from datetime import datetime

input_data = json.load(sys.stdin)
session_id = input_data.get('session_id', 'unknown')
reason = input_data.get('reason', 'unknown')

log_dir = Path('.claude/logs')
log_dir.mkdir(parents=True, exist_ok=True)

log_entry = {
    "session_id": session_id,
    "ended_at": datetime.now().isoformat(),
    "reason": reason
}

with open(log_dir / 'session-history.jsonl', 'a') as f:
    f.write(json.dumps(log_entry) + '\n')

sys.exit(0)
```

### SessionEnd Reasons

The reason field indicates why the session ended:

- clear - User ran /clear

- logout - User logged out

- prompt_input_exit - User exited while prompt was visible

- other - Other exit reasons

## Complete Lifecycle Example

Here's a complete lifecycle configuration:

```
{
  "hooks": {
    "Setup": [
      {
        "matcher": "init",
        "hooks": [
          {
            "type": "command",
            "command": "npm install && echo 'Dependencies installed'"
          }
        ]
      }
    ],

    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "echo '## Context' && git status --short && echo '## Tasks' && cat .claude/tasks/session-current.md 2>/dev/null | head -20"
          }
        ]
      }
    ],

    "PreCompact": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "cp \"$CLAUDE_TRANSCRIPT_PATH\" .claude/backups/last-transcript.jsonl 2>/dev/null || true"
          }
        ]
      }
    ],

    "SessionEnd": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "echo \"Session ended: $(date)\" >> .claude/logs/sessions.log"
          }
        ]
      }
    ]
  }
}
```

## Input Payloads

### SessionStart Input

```
{
  "session_id": "abc123",
  "hook_event_name": "SessionStart",
  "source": "startup",
  "model": "claude-sonnet-4-20250514",
  "cwd": "/path/to/project"
}
```

### Setup Input

```
{
  "session_id": "abc123",
  "hook_event_name": "Setup",
  "trigger": "init",
  "cwd": "/path/to/project"
}
```

### PreCompact Input

```
{
  "session_id": "abc123",
  "hook_event_name": "PreCompact",
  "transcript_path": "~/.claude/projects/.../transcript.jsonl",
  "trigger": "auto",
  "custom_instructions": ""
}
```

### SessionEnd Input

```
{
  "session_id": "abc123",
  "hook_event_name": "SessionEnd",
  "reason": "clear",
  "cwd": "/path/to/project"
}
```

## Best Practices

-

Keep SessionStart fast - It runs every session. Heavy operations go in Setup.

-

Use Setup for one-time work - Dependency installation, migrations, initial setup.

-

Backup before compaction - PreCompact is your last chance to save context.

-

Log session ends - SessionEnd is useful for analytics and debugging.

-

Use matchers wisely - Different behavior for startup vs resume vs compact.

## Next Steps

- Set up the main Hooks Guide for all 12 hooks

- Configure Context Recovery for compaction survival

- Use Stop Hooks for task enforcement

- Explore Skill Activation for automatic skill loading

Last updated on 6/29/2026

Previous

Self-Validating Agents

Next

Context Recovery Hook

### On this page

The Session Lifecycle Hooks

SessionStart: Load Context Every Time

Basic Context Injection

With JSON Output

SessionStart Matchers

Persist Environment Variables

Setup: One-Time Operations

When to Use Setup vs SessionStart

Setup Configuration

PreCompact: Before Context Loss

Backup Transcripts

PreCompact Matchers

Create Recovery Markers

SessionEnd: Cleanup

Log Session Stats

SessionEnd Reasons

Complete Lifecycle Example

Input Payloads

SessionStart Input

Setup Input

PreCompact Input

SessionEnd Input

Best Practices

Next Steps

Stop configuring. Start shipping.Everything you're reading about and more..
Agentic Orchestration Kit for Claude Code.

Get Claude Fast

New

Shopify Kit just dropped

Your in-house Shopify x Claude team for Growth, CRO, Paid ads, retention, SEO, ops and Media gen.

Learn more
