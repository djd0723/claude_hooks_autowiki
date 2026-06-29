---
type: concept
title: "The Agent Manager Role (Who Owns the Harness at Scale)"
created: 2026-06-29
updated: 2026-06-29
tags: [governance, teams, rollout, harness, dri, devex, claudefast]
source_count: 1
sources:
  - sources/clean/claudefa-st-blog-guide-development-agent-manager-role.md
---

# The Agent Manager Role (Who Owns the Harness at Scale)

Once more than a handful of developers run Claude Code, the [harness](./hooks-adoption-ladder.md) itself needs an owner. The **agent manager** is that owner: a hybrid PM/engineer function dedicated to a team's Claude Code ecosystem. The article frames it as a named role Anthropic surfaced in May 2026, drawn from a pattern observed across enterprise rollouts:

> "The rollouts that spread fastest had a dedicated infrastructure investment before broad access."

That "dedicated infrastructure investment" — often a single person — is the agent manager. Crucially, this is about managing the *people and configuration* of a team that uses Claude Code, **not** the multi-agent Agent Teams orchestration feature.

## Why an unowned harness breaks down

Without someone owning the harness, three failure modes surface within the first quarter of broad access:

1. **Tribal knowledge.** "Every developer evolves a personal AI layer" — one engineer's slick skills, another's private MCP server, a third's bug-catching stop hook — and none of it is shared. New hires get the bare default install and rediscover the same lessons from scratch.
2. **Inconsistent results.** Same model, same codebase, very different output quality, because "the harness determines performance more than the model" and an unstandardized harness varies wildly per developer. The gap widens with [agent teams](./forked-subagents.md): "a shared, well-built harness is what makes multi-agent runs reproducible instead of one engineer's lucky configuration."
3. **Security drift.** [Permissions](./permission-evaluation.md) get configured ad hoc; MCP servers connect to internal systems with whatever scope the developer granted. "Infosec finds out three months later that a credential is in plaintext inside an .mcp.json file checked into a personal fork." This is "the failure mode that gets AI tools banned by the platform team."

> "Any one is fixable. All three together is the kind of mess that takes a quarter to clean up and a year to recover trust from."

## What the role owns: five areas

The job maps onto the seven pieces of the harness with a governance lens:

- **The plugin marketplace.** [Plugins](./plugin-distribution.md) are "how good harness configuration moves from one machine to the entire org." The agent manager runs it "the same way an internal package registry is run" — new plugins go through review, updates land on a schedule, deprecations are announced.
- **Approved skills.** [Skills](./skills.md) are the workflow library. Anthropic's recommendation is direct: *"starting with a defined set of approved skills, required code review processes, and limited initial access, and expand as confidence builds."* The manager decides what gets promoted from a personal fork into the org-wide bundle.
- **CLAUDE.md conventions.** The manager owns the canonical [`CLAUDE.md`](./sdk-project-instructions.md) hierarchy. The [self-improving CLAUDE.md pattern](./self-improving-claude-md.md) is cited as one tool here — "a stop hook that proposes updates to the agent manager's review queue instead of letting rules go stale."
- **Permissions policy.** The infosec touchpoint: deciding "what tools are allowed, what commands need approval, what gets denied at the [settings](./settings-files.md) level." Output is an org-standard [`.claude/settings.json`](./permission-settings.md) plus written policy for what developers can and cannot relax.
- **The configuration review cadence.** Anthropic suggests "a meaningful configuration review every three to six months, and also after major model releases when performance feels like it has plateaued." The manager schedules it, gathers telemetry, and retires "rules that have become constraints rather than guardrails."

## Why hybrid PM/engineer

"Pure engineering and pure product management both fail at it." A pure engineer "builds beautiful internal tooling that nobody adopts because the rollout was never planned"; a pure PM "ships a glossy adoption plan but cannot debug a misconfigured hook on a Friday afternoon."

The **technical floor**: write a hook, read a `settings.json`, debug a silently failing MCP server, reason about [which skill should activate when](./skill-activation.md), and "understand the difference between a permission rule and an MCP scope." The **product floor**: define rollout phases, write developer-facing docs, run office hours, and "make calls on what gets prioritized when ten developers want ten incompatible features." Most managers grow into one side from the other.

## Minimum viable version: the DRI pattern

Most orgs do not need a dedicated team on day one. Anthropic's minimum viable version is a single DRI:

> "one person with ownership over the Claude Code configuration, the authority to make calls on settings, permissions policy, the plugin marketplace, and CLAUDE.md conventions, and the responsibility to keep them current."

This works for orgs under fifty engineers. The DRI specifically owns: the canonical `CLAUDE.md` and subdirectory files; the org-standard `.claude/settings.json`; the approved-skills list and proposal process; the plugin marketplace ("even if it is just a private git repo with three plugins in it"); and the escalation path. The DRI *coordinates with but does not own* infosec policy review, the Anthropic vendor relationship, and broad developer training — "the DRI is the interface."

## Where it sits organizationally

The role sits "under developer experience or developer productivity" — DevEx is the natural home "because the work is fundamentally about making developers faster and more consistent." Platform Engineering or Internal Tools are valid alternatives. Anthropic explicitly endorses a cross-functional working group for larger orgs:

> "We've observed the smoothest deployments at organizations that establish cross-functional working groups early by bringing together engineering, information security, and governance representatives to define requirements together and build a rollout roadmap."

## The first 90 days

A phased plan that "repeats whether you are the DRI for a thirty-engineer startup or one of two engineers at a multi-thousand-engineer org":

- **Days 1–30 — Audit and baseline.** "Not building." Inventory every personal `.claude/` setup, pull telemetry (sessions/week, models, context usage, failure modes), and "talk to infosec — get current concerns on the table before you ship anything." A Slack search for "Claude Code" over the last quarter is "high-signal; the complaints are your roadmap." End with a written audit, three-to-five problems to solve first, a stakeholder map, and a draft charter.
- **Days 31–60 — Build the v1 harness.** Assemble every AI-layer component: a layered `CLAUDE.md`, a starter skills set, the hooks that enforce repeatedly-violated conventions, the MCP servers developers actually use, and a [plugin](./plugin-components.md) that bundles it. Write **two pages** of docs, not twenty — "if you cannot fit it on two pages, the harness is too complicated for v1." Dogfood it yourself for a full week first.
- **Days 61–90 — Quiet rollout to one team.** "The discipline is to resist shipping to everyone at once." Pick one enthusiastic team, be present in their Slack, fix what breaks, ship updates weekly. By day 90: a battle-tested v1, a v2 change list from real usage, a rollout plan for the next teams, and a governance doc.

## Governance recommendations

Drawn from practice and Anthropic's explicit advice:

- "Start with limited initial access and expand as confidence builds." Limited access is what "lets you fix problems before they become company-wide incidents."
- Maintain a defined set of approved skills "with a written promotion process" — lightweight is fine, "but it has to exist."
- Require [code review](./hooks-adoption-ladder.md) on anything in `.claude/` that ships with the org-standard repo: "The harness is code. Treat it like code."
- Stand up the cross-functional working group within six months; document the escalation path.

## Hiring or promoting

"Most agent managers in 2026 are promoted internally" — the role is too new for a deep external pool, and internal candidates have the org context. Signals: "writes good documentation unprompted, has a .claude/ folder other engineers have asked to borrow from, runs informal office hours about AI tooling without being asked, has shipped at least one piece of internal infrastructure that other teams adopted." For external hires the rubric "is closer to a senior platform engineer than a traditional PM," and compensation "lands between senior engineer and senior PM."

## Why it matters

> "The seven pieces of the harness need an owner. Without one, the harness becomes the bottleneck instead of the leverage point. With one, the harness compounds."

The article predicts "agent manager or an equivalent title" will appear in org charts by the end of 2026, and that orgs which resist "will end up creating it under a different name twelve months later, after the second or third incident that traces back to nobody owning the harness."

## Related concepts

- [Hooks Adoption Ladder](./hooks-adoption-ladder.md) — the maturity progression of the harness the manager owns
- [Plugin Distribution](./plugin-distribution.md) — the mechanism agent managers use to ship the harness org-wide
- [Skills](./skills.md) — the curated workflow library the manager governs
- [Self-Improving CLAUDE.md](./self-improving-claude-md.md) — a tool for keeping CLAUDE.md conventions current via a review queue
- [Permission Evaluation](./permission-evaluation.md) — the permissions-policy surface that is the infosec touchpoint
- [Settings Files](./settings-files.md) — the `.claude/settings.json` the manager standardizes
