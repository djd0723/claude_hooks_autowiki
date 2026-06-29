---
type: comparison
title: "CLAUDE.md rules vs. skills workflows"
created: 2026-06-29
updated: 2026-06-29
tags: [skills, claude-md, rules, workflows, progressive-disclosure, listing-budget, decision]
sources:
  - sources/clean/claudefa-st-blog-guide-mechanics-path-scoped-skills.md
---

# CLAUDE.md rules vs. skills workflows

Before you decide *how* a [skill](../concepts/skills.md) activates, there's a prior question that
the [path-scoped-skills](../concepts/path-scoped-skills.md) source calls "the single most common
skill mistake": should this knowledge be a skill at all, or a [CLAUDE.md](../concepts/sdk-project-instructions.md)
rule? They feel interchangeable — both are markdown you write to steer Claude — but they load on
opposite schedules, and putting knowledge in the wrong one means you "pay twice." This page maps
the distinction and the decision tree.

## The distinction: always-on rules vs. on-demand procedures

The source draws the line in two sentences:

> "CLAUDE.md holds rules. Conventions that must hold every time Claude touches the relevant area.
> Loaded automatically, always in context."

> "Skills hold workflows. Multi-step procedures that fire only on specific task types. Loaded on
> demand."

The whole difference is the loading schedule. A CLAUDE.md rule is **always present** — it costs
context in every session whether or not it's relevant, which is the price of guaranteeing it's
never missed. A skill is **loaded on demand** — it costs nothing until its [activation](./description-driven-activation-vs-activation-hook.md)
fires, which is the price of it sometimes not firing when you wanted it to.

| | CLAUDE.md rule | Skill workflow |
| :-- | :-- | :-- |
| **Holds** | Conventions that must always hold | Multi-step procedures for specific tasks |
| **Loaded** | Automatically, every session | On demand, when activation fires |
| **Cost** | Context in every session | Nothing until invoked |
| **Risk** | Bloats sessions where it's irrelevant | Missed on tasks where it never activates |
| **Reach for when** | "Every-time convention" | "Specific-task procedure" |

## The "pay twice" failure — both directions

Get the mapping backwards and you lose either way:

> "Rules in skills means Claude misses the rule on the 80% of tasks where the skill never
> activates. Workflows in CLAUDE.md means the procedure burns tokens in every session even when
> irrelevant, the bloat problem the skill listing budget was added to police."

So the two failure modes are symmetric:

- **Rule misplaced as a skill** → it silently doesn't apply on the majority of tasks, because the
  skill never activated. A convention that holds only 20% of the time isn't a convention.
- **Workflow misplaced in CLAUDE.md** → it burns tokens in every session even when irrelevant —
  exactly the bloat the [skill listing budget](../concepts/skill-invocation-control.md) exists to
  police.

## The decision tree

The source gives a three-way split — and note that *both* can be path-scoped, so the question is
never "global vs. local," it's "rule vs. workflow":

- **Every-time convention** → it's a rule. Put it in a layered CLAUDE.md.
- **Specific-task procedure** (adding a route, reviewing money code, writing a migration) → it's
  a workflow. Put it in a skill.
- **Both** → split it. *"The convention goes in CLAUDE.md, the procedure goes in a skill, and the
  skill references the CLAUDE.md rule."*

That third branch is the important one: many real cases are a convention *plus* a procedure, and
the right move is not to pick one container but to split the knowledge across both and cross-link
them — the rule stays always-on, the procedure stays on-demand, and the skill points back at the
rule so the procedure carries its own constraint.

## Path scoping narrows, it doesn't decide

It's tempting to think [path scoping](../concepts/path-scoped-skills.md) replaces this decision —
"just scope the skill to the directory." It doesn't. Scoping changes *visibility* (a skill only
enters discovery when work touches a matching path); it does not change *schedule* (a scoped skill
still loads on demand, not always). A path-scoped CLAUDE.md and a path-scoped skill are both
possible — so a convention that should hold "every time Claude touches `services/billing`" is
still a *rule* and belongs in a layered CLAUDE.md, not a billing-scoped skill. Scoping shrinks
the candidate set; it never converts a workflow into a convention or vice-versa.

## Decision table

| Situation | Reach for |
| :-------- | :-------- |
| Convention that must hold every time | **CLAUDE.md rule** |
| Multi-step procedure for a specific task type | **Skill workflow** |
| Both a convention and a procedure | **Split: rule in CLAUDE.md, procedure in skill, skill references the rule** |
| Knowledge relevant only under one directory | **Path-scope it — but still as the right *kind*** (rule vs. workflow) |
| Procedure burning tokens in every session | **Move it out of CLAUDE.md into a skill** |
| Convention missed on most tasks | **Move it out of a skill into CLAUDE.md** |

## Related

- [Path-Scoped Skills](../concepts/path-scoped-skills.md) — the `paths:` glob and the rules-vs-workflows mental model this page expands
- [Skills](../concepts/skills.md) — the core concept; what a skill is and how it loads
- [SDK Project Instructions](../concepts/sdk-project-instructions.md) — how CLAUDE.md and other always-on instruction sources load
- [Skill Invocation Control](../concepts/skill-invocation-control.md) — the listing budget that "polices" CLAUDE.md/skill bloat
- [Description-driven activation vs. the skill-activation hook](./description-driven-activation-vs-activation-hook.md) — once it *is* a skill, how to make it fire
- [Skill Content Lifecycle](../concepts/skill-content-lifecycle.md) — the progressive-disclosure discipline that keeps on-demand loading cheap
