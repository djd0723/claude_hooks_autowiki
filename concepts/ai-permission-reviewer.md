---
type: concept
title: "AI Permission Reviewer (ClaudeFast Permission Hook Case Study)"
created: 2026-06-29
updated: 2026-06-29
tags: [hooks, permissions, case-study, claudefast, llm-evaluator, latency, cost]
source_count: 1
sources:
  - sources/clean/claudefa-st-blog-tools-hooks-permission-hook-guide.md
---

# AI Permission Reviewer (ClaudeFast Permission Hook Case Study)

The [permission evaluation](./permission-evaluation.md) reference describes how Claude Code resolves **allow/ask/deny** from your rules, and the [PermissionRequest recipes](./hook-automation-use-cases.md) note that a hook can "approve or deny programmatically instead of showing a dialog." This page is grounded evidence of one third-party product — ClaudeFast's *Permission Hook* (`@abdo-el-mobayad/claude-code-fast-permission-hook`, CLI `cf-approve`) — that builds an entire workflow around exactly that hook point. It's a commercial kit; what's transferable is its **architecture**, not its branding.

## The problem it targets: the permission dilemma

The pitch frames vanilla Claude Code as two bad options:

> "Option 1: Click approve constantly. Safe, but flow-destroying. Complex features mean 50+ permission prompts."

> "Option 2: Use --dangerously-skip-permissions. Fast, but terrifying. One hallucinated rm -rf / and your system is gone."

The product positions itself as a middle path:

> "The Permission Hook gives you a third option: intelligent delegation. Claude runs without interruption. Dangerous commands get blocked automatically. Edge cases go to a fast LLM for context-aware decisions."

This is the same deterministic-guarantee mindset behind the [hooks adoption ladder](./hooks-adoption-ladder.md), applied specifically to the permission surface: rather than trusting the model to skip its own prompts, a hook intercepts every request and decides.

## The central pattern: a three-tier decision funnel

The transferable design is a **cost/latency cascade** — cheap deterministic checks first, an LLM call only for the genuine ambiguities:

- **Tier 1 — Fast Approve (no AI).** A hard-coded allowlist of safe tools passes through immediately: "Read, Glob, Grep, WebFetch, WebSearch", "Write, Edit, MultiEdit, NotebookEdit", "TodoWrite, Task, all MCP tools." The stated payoff: "No latency. No cost. Claude keeps working."
- **Tier 2 — Fast Deny (no AI).** A hard-coded denylist blocks catastrophe instantly — `rm -rf /`, `git push --force origin main`, `mkfs /dev/sda`, the `:(){ :|:& };:` fork bomb. "No AI evaluation needed. Hard-coded rules protect you from catastrophic mistakes."
- **Tier 3 — LLM Analysis (cached).** Only "ambiguous operations" reach a "fast, cheap LLM (GPT-4o-mini via OpenRouter)," which receives the tool, command, working directory, and recent context:

```json
{
  "tool": "Bash",
  "command": "docker system prune -af",
  "working_directory": "/home/user/project",
  "recent_context": "User asked to clean up Docker resources"
}
```

> "The LLM sees what you're trying to accomplish and makes an intelligent decision. Decisions are cached - repeat the same command and it's instant."

**Takeaway:** an LLM-evaluator permission hook only stays cheap and fast if most calls never reach the LLM. The deterministic allow/deny tiers aren't a fallback — they're the whole economic argument. The same allow-broadly / deny-specifically split appears natively in [permission evaluation](./permission-evaluation.md) ("run all Bash commands without prompts except for a few you want blocked"); this product layers an AI tier behind it for the residual gray area.

## Why caching matters: the cost model

The product reports its LLM tier costs "roughly $1 per 5,000+ LLM decisions" and adds:

> "In practice, most operations hit Tier 1 or 2, so a dollar lasts months."

Two reusable lessons fall out:

- **Cache by command identity.** A permission decision for an identical `(tool, command)` is stable, so a TTL'd cache (the config ships `ttlHours: 168` — one week) turns repeat operations into instant Tier-1-speed hits.
- **Budget the AI tier as the exception.** Because deterministic tiers absorb the bulk of traffic, the per-decision LLM cost is amortized to near-zero — the same "guard hooks are cheap, context/LLM hooks have recurring cost" economics surfaced in the [ClaudeKit case study](./production-hook-patterns.md).

## How it wires in: the `PermissionRequest` hook

The installer adds a single [`command` hook](./hook-types.md) matching every tool on the `PermissionRequest` event:

```json
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

This is the concrete form of the [PermissionRequest automation recipe](./hook-automation-use-cases.md) ("Auto-approve Bash commands that match a whitelist … without prompting"). It can be installed at **device level** (`~/.claude/settings.json`, "applies everywhere") or **project level** (`.claude/settings.local.json`, "project-specific rules") — the same [user-vs-project scope](./hook-scope.md) choice every hook faces. Its own LLM config lives outside Claude's settings, at `~/.claude-code-fast-permission-hook/config.json` (provider, model, API key, cache TTL).

## Caveats worth flagging

- **The decision authority is a separate third-party LLM**, not Claude — by default GPT-4o-mini via OpenRouter, with the user's own API key. Approval quality and privacy (the command and "recent context" are sent to that provider) depend on a model and vendor outside Claude Code.
- **Tiers 1–2 are hard-coded lists.** They are only as good as their coverage; a destructive command not on the denylist falls through to the LLM's judgement, and a novel-but-safe tool not on the allowlist incurs an LLM call.
- **It markets eliminating `--dangerously-skip-permissions`,** but the net trust model is "delegate approvals to an automated reviewer" — a different risk posture, not zero risk. Compare the native, model-free controls in [permission modes](./permission-modes.md) and [permission settings](./permission-settings.md).

## Related concepts

- [Permission Evaluation](./permission-evaluation.md) — the native allow/ask/deny algorithm this product sits in front of
- [Hook Automation Use Cases](./hook-automation-use-cases.md) — the generic `PermissionRequest` recipes this is one concrete instance of
- [Production Hook Patterns (ClaudeKit Case Study)](./production-hook-patterns.md) — the sibling third-party case study, with the same "deterministic guards are cheap, LLM tiers have cost" lesson
- [Hook Decision Control](./hook-decision-control.md) — the allow/deny mechanics a `PermissionRequest` hook returns
- [Hook Types](./hook-types.md) — the `command` handler the installer wires
- [Hook Scope](./hook-scope.md) — the device-vs-project install choice
- [Permission Modes](./permission-modes.md) — the native alternative to `--dangerously-skip-permissions`
- [Hooks Adoption Ladder (Practitioner Playbook)](./hooks-adoption-ladder.md) — the "guarantees, not suggestions" framing this product applies to permissions
- [Permission rules vs. permission hooks](../comparisons/permission-rules-vs-permission-hooks.md) — the rules-vs-hook decision this case study is the worked example of
