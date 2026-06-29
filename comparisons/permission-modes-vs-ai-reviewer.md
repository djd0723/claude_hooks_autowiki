---
type: comparison
title: "Switching permission modes vs. an AI permission reviewer"
created: 2026-06-29
updated: 2026-06-29
tags: [permissions, modes, hooks, prompt-fatigue, ergonomics, automation, cost, privacy]
sources:
  - sources/clean/claudefa-st-blog-guide-development-permission-management.md
  - sources/clean/claudefa-st-blog-tools-hooks-permission-hook-guide.md
---

# Switching permission modes vs. an AI permission reviewer

Both of these target the **same pain**, stated almost identically by two different sources:

> "Problem: Claude Code asking for permission on every file edit and command kills your flow and burns time."

> "Option 1: Click approve constantly. Safe, but flow-destroying. Complex features mean 50+ permission prompts."

And both reject the same naive escape — blanket `bypassPermissions` / `--dangerously-skip-permissions`, "Fast, but terrifying." Where they diverge is *what* they automate to get out of the dilemma. The [mode playbook](../concepts/permission-mode-strategies.md) automates your **posture** — it picks a coarse allow/ask stance for a whole phase of work and lets you flip it by hand. The [AI permission reviewer](../concepts/ai-permission-reviewer.md) automates the **decision** — it leaves a single per-call hook in place that judges each request as it arrives. One is a dial you turn; the other is a delegate you hire.

## The one axis that separates them: posture vs. decision

| | **Permission mode switching** | **AI permission reviewer hook** |
| :-- | :-- | :-- |
| What's automated | The **posture** (a session-wide allow/ask stance) | The **decision** (each individual call) |
| Granularity | Coarse — one mode covers the whole session/phase | Fine — every `(tool, command)` judged on its own |
| Who decides | **You**, by choosing the mode for the phase | A **separate LLM** (default GPT-4o-mini via OpenRouter) for the gray area |
| Mechanism | Native, model-free; `Shift+Tab` cycle or `defaultMode` | A `PermissionRequest` `command` hook (`cf-approve`) |
| Cost / latency | None ("No latency. No cost.") | LLM tier only — "roughly $1 per 5,000+ LLM decisions," cached `ttlHours: 168` |
| Privacy | Stays local | Command + "recent context" sent to a third-party provider |
| Trigger to act | Manual — you must remember to switch | Automatic — fires on every tool call |
| Failure mode | Wrong mode left on (too loose / too tight) | Denylist gap, or the external model misjudges |

## Why mode switching is coarse — and that's the point

The mode playbook's whole value is that it makes the cost/safety tradeoff *once per phase* instead of once per prompt. It maps a phase of work to a stance:

> "Five permission modes for five development scenarios. `default` for safety, `acceptEdits` for productivity, `plan` for exploration, `dontAsk` for automation, and `bypassPermissions` for isolated environments. Match the mode to your current needs, and layer sandboxing on top for defense-in-depth."

Cycling is a keystroke — `normal → auto-accept edits → plan mode → normal` on `Shift+Tab` — or a one-line `{"defaultMode": "acceptEdits"}` to start a session somewhere. The cost of coarseness is that *you* are the trigger: the playbook's own pitfalls are "Mode confusion" ("Check your current mode before starting work") and the twin errors of leaving it too loose (over-permissioning) or too tight (under-permissioning, "Constantly clicking 'Allow' defeats the purpose"). The dial only helps if you keep turning it.

## Why the AI reviewer is fine-grained — and what that costs

The reviewer never asks you to switch anything; it judges each call through a three-tier funnel — deterministic **Fast Approve** and **Fast Deny** lists first, an LLM only for "ambiguous operations":

> "The LLM sees what you're trying to accomplish and makes an intelligent decision. Decisions are cached - repeat the same command and it's instant."

That fineness buys context-awareness a mode can't have: `docker system prune -af` can be approved in a cleanup context and questioned elsewhere, because the hook receives the tool, command, working directory, and recent context. The price is a different trust posture, not zero risk — **the decision authority is a separate third-party LLM**, the command and its context leave your machine, and the deterministic tiers "are only as good as their coverage."

## The floor they share

Neither one widens what the [permission system](../concepts/permission-evaluation.md) already forbids. The mode playbook says so directly — "deny rules always win" — so a looser mode only moves *within* the deny-bounded envelope. And because the reviewer is implemented as a hook, it inherits the [one-way-valve asymmetry](./permission-rules-vs-permission-hooks.md): a `PermissionRequest` hook can only narrow or judge, never rescue a call a `deny` rule kills. So both tools operate strictly *above* the static `deny` floor — they decide among the calls the rules would already have asked about, and nothing below that line.

This also makes them **composable, not exclusive**: run `dontAsk` (a mode) with explicit allow rules for a safe hands-off baseline, and let an AI reviewer hook adjudicate the residual gray area — the deny floor still bounds both.

## When to reach for each

**Reach for mode switching when:**

- You're an interactive human at the keyboard moving through phases — explore in `plan`, build in `acceptEdits`, drop to `default` for the risky parts.
- You want **zero added cost, latency, privacy surface, or dependency** — it's native and model-free.
- A coarse, session-wide stance is good enough and you're willing to be the one who flips it.

**Reach for an AI reviewer hook when:**

- No human is watching, or the prompt volume is too high to triage by hand ("50+ permission prompts").
- The right answer genuinely **depends on per-call context** a mode can't read (intent, cwd, recent conversation).
- You accept the tradeoff: recurring (if small) LLM cost, added latency on the gray-area tier, and delegating decision authority + context to an external model/vendor.

**Use `dontAsk` instead of either** when you want hands-off automation without an external model: it "auto-denies anything not pre-approved via `/permissions` or `allow` rules" — the safe middle the mode playbook calls out as the under-used ground between approving everything by hand and bypassing everything.

## Related

- [Permission Mode Strategies](../concepts/permission-mode-strategies.md) — the practitioner playbook for matching mode to phase
- [AI Permission Reviewer](../concepts/ai-permission-reviewer.md) — the three-tier funnel case study this compares against
- [Permission rules vs. permission hooks](./permission-rules-vs-permission-hooks.md) — the deeper rules-vs-hook asymmetry both sit on top of
- [Permission Modes](../concepts/permission-modes.md) — the reference for what each mode does
- [Permission Evaluation](../concepts/permission-evaluation.md) — the deny→ask→allow floor neither tool can widen
- [Hooks Adoption Ladder](../concepts/hooks-adoption-ladder.md) — the "guarantees, not suggestions" framing the reviewer applies to permissions
