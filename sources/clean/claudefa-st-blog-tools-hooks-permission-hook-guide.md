---
type: source
source_url: https://claudefa.st/blog/tools/hooks/permission-hook-guide
title: "Claude Code Permission Hook: Skip Prompts Safely"
raw_capture: ../raw/claudefa-st-blog-tools-hooks-permission-hook-guide.html
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

# Claude Code Permission Hook: Uninterrupted Coding Without the Risk

Run Claude Code without constant permission prompts or dangerous skip flags. Delegate approvals to an AI reviewer that understands context.

Stop configuring. Start shipping.Everything you're reading about and more..
Agentic Orchestration Kit for Claude Code.

Get Claude Fast

Problem: Every time Claude wants to read a file or run a command, you're clicking "approve." Twenty clicks later, you've lost your flow state and forgotten what you were building.

Quick Win: Install the Permission Hook and never click approve again:

```
npm install -g @abdo-el-mobayad/claude-code-fast-permission-hook
cf-approve install
cf-approve config
```

Three commands. Now Claude runs uninterrupted while dangerous operations get blocked automatically. No --dangerously-skip-permissions required.

## The Permission Dilemma

You have two bad options with vanilla Claude Code:

Option 1: Click approve constantly. Safe, but flow-destroying. Complex features mean 50+ permission prompts. You lose context. You lose momentum. You lose the magic of AI-assisted coding.

Option 2: Use --dangerously-skip-permissions. Fast, but terrifying. One hallucinated rm -rf / and your system is gone. Fine for throwaway projects. Unacceptable for real work.

The Permission Hook gives you a third option: intelligent delegation. Claude runs without interruption. Dangerous commands get blocked automatically. Edge cases go to a fast LLM for context-aware decisions.

## How It Works: Three-Tier Decision System

When Claude requests permission, the hook evaluates it instantly:

Tier 1 - Fast Approve (No AI Needed)

Safe operations pass through immediately:

- Read, Glob, Grep, WebFetch, WebSearch

- Write, Edit, MultiEdit, NotebookEdit

- TodoWrite, Task, all MCP tools

No latency. No cost. Claude keeps working.

Tier 2 - Fast Deny (No AI Needed)

Dangerous operations get blocked instantly:

```
# These never execute, period
rm -rf /                      # System destruction
git push --force origin main  # Protected branch overwrite
mkfs /dev/sda                 # Disk formatting
:(){ :|:& };:                  # Fork bomb
```

No AI evaluation needed. Hard-coded rules protect you from catastrophic mistakes.

Tier 3 - LLM Analysis (Cached)

Ambiguous operations get sent to a fast, cheap LLM (GPT-4o-mini via OpenRouter) for context-aware evaluation:

```
{
  "tool": "Bash",
  "command": "docker system prune -af",
  "working_directory": "/home/user/project",
  "recent_context": "User asked to clean up Docker resources"
}
```

The LLM sees what you're trying to accomplish and makes an intelligent decision. Decisions are cached - repeat the same command and it's instant.

## Configuration

The hook stores settings at ~/.claude-code-fast-permission-hook/config.json:

```
{
  "llm": {
    "provider": "openai",
    "model": "openai/gpt-4o-mini",
    "apiKey": "sk-or-v1-your-key",
    "baseUrl": "https://openrouter.ai/api/v1"
  },
  "cache": {
    "enabled": true,
    "ttlHours": 168
  }
}
```

OpenRouter is recommended for best latency. Get your key at openrouter.ai. Cost: roughly $1 per 5,000+ LLM decisions. In practice, most operations hit Tier 1 or 2, so a dollar lasts months.

## Installation Levels

Device Level (recommended): Configure once in ~/.claude/settings.json, applies everywhere. Set it and forget it.

Project Level: Configure in .claude/settings.local.json for project-specific rules.

The installer adds this to your settings:

```
{
  "hooks": {
    "PermissionRequest": [
      {
        "matcher": "*",
        "hooks": [
          {
            "type": "command",
            "command": "cf-approve permission"
          }
        ]
      }
    ]
  }
}
```

## When Things Go Wrong

Error: "Permission denied" on all operations

Fix: Your API key is missing or invalid:

```
cf-approve config
```

Re-enter your OpenRouter key.

Error: "Hook not triggering"

Fix: Verify installation:

```
cf-approve doctor
cf-approve status
```

Behavior seems inconsistent

Fix: Clear the decision cache:

```
cf-approve clear-cache
```

## The Two Foundational Hooks

The Permission Hook is one half of ClaudeFast's hook philosophy. The other half is the Skill Activation Hook, which ensures Claude loads the right skills at the right time.

Together, they create friction-free development: you type naturally, Claude works efficiently, and the framework handles orchestration invisibly.

## You Can Now Code Without Interruption

- You just installed context-aware permission automation

- Set up the main Hooks Guide for complete hook coverage

- Configure the Stop Hook to ensure task completion

- Try Context Recovery to survive compaction

- Go deeper: Explore skills for specialized agent workflows

No more permission fatigue. No more dangerous flags. Just Claude doing what Claude does best - building your software while you focus on the big picture.

Last updated on 6/29/2026

Previous

Skill Activation Hook

Next

Best Skills Repos

### On this page

The Permission Dilemma

How It Works: Three-Tier Decision System

Configuration

Installation Levels

When Things Go Wrong

The Two Foundational Hooks

You Can Now Code Without Interruption

Stop configuring. Start shipping.Everything you're reading about and more..
Agentic Orchestration Kit for Claude Code.

Get Claude Fast

New

Shopify Kit just dropped

Your in-house Shopify x Claude team for Growth, CRO, Paid ads, retention, SEO, ops and Media gen.

Learn more
