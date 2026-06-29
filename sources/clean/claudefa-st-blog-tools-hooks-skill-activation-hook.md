---
type: source
source_url: https://claudefa.st/blog/tools/hooks/skill-activation-hook
title: "Claude Code Skill Hook: Guarantee 100% Loading"
raw_capture: ../raw/claudefa-st-blog-tools-hooks-skill-activation-hook.html
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

# Claude Code Skill Activation Hook: Guarantee 100% Skill Loading

Stop Claude from ignoring your skills. The Skill Activation Hook appends recommendations to prompts before Claude sees them. Zero memory reliance.

Stop configuring. Start shipping.Everything you're reading about and more..
Agentic Orchestration Kit for Claude Code.

Get Claude Fast

Problem: You tell Claude Code to load a skill. It forgets. You add instructions to CLAUDE.md. It ignores them. You end up manually reminding Claude about skills that should be automatic.

Quick Win: The Skill Activation Hook intercepts your prompts and appends skill recommendations before Claude sees the message. Claude can't forget because it never had to remember.

When you type "help me implement a feature", Claude actually sees:

```
help me implement a feature

SKILL ACTIVATION CHECK

CRITICAL SKILLS (REQUIRED):
  -> session-management

RECOMMENDED SKILLS:
  -> git-commits

ACTION: Use Skill tool BEFORE responding
```

Claude now knows exactly which skills to load. No guessing. No forgetting.

## How It Works

The hook uses Claude Code's UserPromptSubmit event. Every prompt you send triggers this flow:

- You type a message - Your natural language request

- Hook intercepts - Before Claude sees anything

- Pattern matching - Hook checks skill-rules.json for keyword and intent matches

- Append recommendations - Matching skills get added to your message

- Claude receives both - Your prompt plus skill guidance

The hook runs in milliseconds. You won't notice any delay.

## The Matching System

Two strategies work together:

Keyword Matching - Simple string matching. If your prompt contains "commit" or "git push", the git-commits skill triggers.

Intent Patterns - Regex for natural language variation. A pattern like (implement|build).*?feature catches "let's implement this feature" and "build a new feature for me".

## Configuration: skill-rules.json

Every skill has triggers defined in .claude/skills/skill-rules.json:

```
{
  "skills": {
    "session-management": {
      "enforcement": "suggest",
      "priority": "critical",
      "promptTriggers": {
        "keywords": ["feature", "implement", "build", "refactor"],
        "intentPatterns": ["(implement|build).*?feature"]
      }
    },
    "git-commits": {
      "enforcement": "suggest",
      "priority": "high",
      "promptTriggers": {
        "keywords": ["commit", "git push", "commit changes"],
        "intentPatterns": ["(create|make).*?commit"]
      }
    }
  }
}
```

Priority levels control how the hook groups suggestions:

- Critical - Must load before any work

- High - Strongly recommended

- Medium - Helpful context

- Low - Optional enhancement

## Customization for Your Speech Patterns

The hook adapts to how you talk. If you always say "push my code" instead of "git push", add it:

```
"keywords": ["commit", "git push", "push my code", "commit changes"]
```

After creating any new skill, update its triggers in skill-rules.json. Then read those triggers and keep those speech patterns in mind when prompting.

## Session Intelligence

The hook tracks what it already recommended. If it suggested session-management earlier in your conversation, it won't repeat the suggestion. Less noise, same coverage.

Session state lives in recommendation-log.json and auto-cleans after 7 days.

## Setup in ClaudeFast

The hook is pre-configured. Verify your .claude/settings.local.json includes:

Windows:

```
{
  "hooks": {
    "UserPromptSubmit": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "cmd /c \".claude\\hooks\\SkillActivationHook\\skill-activation-prompt.cmd\""
          }
        ]
      }
    ]
  }
}
```

Linux/Mac:

```
{
  "hooks": {
    "UserPromptSubmit": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "bash .claude/hooks/SkillActivationHook/skill-activation-prompt.sh"
          }
        ]
      }
    ]
  }
}
```

## Common Issues

No suggestions appearing
Check that your keywords match your actual speech patterns. Run the hook manually to test:

```
echo '{"session_id":"test","prompt":"implement a feature"}' | node .claude/hooks/SkillActivationHook/skill-activation-prompt.mjs
```

Suggestions appearing when not needed
Your keywords may be too broad. Use more specific terms or intent patterns.

Duplicate suggestions
The hook might be configured in both global and project settings. Keep it in one location only.

## Next Actions

- Check your skill-rules.json matches your vocabulary

- Add keywords for new skills you create

- Set up the main Hooks Guide for complete hook coverage

- Configure the Stop Hook to enforce task completion

- Learn more about CLAUDE.md configuration to complement the hook

- Review the skills guide if you need to create new skills

The Skill Activation Hook removes human memory from the equation. You focus on describing what you need. The framework handles which skills Claude should use. That's the point of a framework.

Last updated on 6/29/2026

Previous

Context Recovery Hook

Next

Permission Hook

### On this page

How It Works

The Matching System

Configuration: skill-rules.json

Customization for Your Speech Patterns

Session Intelligence

Setup in ClaudeFast

Common Issues

Next Actions

Stop configuring. Start shipping.Everything you're reading about and more..
Agentic Orchestration Kit for Claude Code.

Get Claude Fast

New

Shopify Kit just dropped

Your in-house Shopify x Claude team for Growth, CRO, Paid ads, retention, SEO, ops and Media gen.

Learn more
