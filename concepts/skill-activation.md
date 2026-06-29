---
type: concept
title: "Skill Activation (Description-Driven)"
created: 2026-06-29
updated: 2026-06-29
tags: [skills, activation, description, intent-classifier, failure-modes, practitioner-pattern]
source_count: 1
sources:
  - ../sources/clean/claudefa-st-blog-guide-mechanics-claude-skills-guide.md
---

# Skill Activation (Description-Driven)

Once [path scoping](./path-scoped-skills.md) and [discovery](./skill-discovery.md) decide *which* skills are even visible, the model still has to decide *which one to fire* on a given prompt. That decision is driven by the `description` field, not the skill's name:

> "Activation is description-driven, not name-driven."

This is the model-side counterpart to the [Skill Activation Hook](./skill-activation-hook.md), which removes the model's judgment from the loop entirely. Understanding the model-side flow is what tells you how to write a `description` that actually fires.

## The decision flow

On every message, Claude runs the visible skills through four steps:

- **Per-prompt scan.** Claude reads the metadata (name + description) for every registered skill — a cheap pass that costs only the [listing budget](./skill-invocation-control.md), not the full bodies.
- **Description matching.** Claude semantically matches your prompt against each `description`. "The name alone almost never triggers activation; the description does."
- **Disambiguation.** When two skills could both apply, Claude breaks the tie toward specificity:

  > "When two skills could plausibly apply, Claude either asks you to confirm, or picks the one with the more specific description. This is why vague descriptions like 'helps with documents' rarely fire — they get beaten by anything more specific."

- **Load and execute.** On a match, the full [`SKILL.md` body loads into context](./skill-content-lifecycle.md) and is treated as authoritative for the rest of that task.

## The description is an intent classifier

The single most useful framing from this source — your `description` is not marketing copy, it is the classifier that decides whether the skill ever runs:

> "Your description field is doing the work of an intent classifier. Write it for that job. Mention the trigger phrases users actually type. Mention the file types or task types the skill applies to. Skip marketing language."

The concrete contrast:

> "A description like 'Reviews staged or modified code for security flaws. Triggers on phrases like 'review my changes' or 'security check'' will activate reliably. A description like 'Helps you with code' will not."

This is the same lesson [path scoping](./path-scoped-skills.md) reaches from the other direction — scoping shrinks the candidate set, but "the description still has to win against the others in that set." Both layers fail the same way when the description is vague.

## Real failure modes

Beyond a weak description, three activation failures recur once a project has more than a handful of skills:

- **The Silent No-Op.** The skill loads, runs, and returns nothing useful — "almost always a missing `allowed-tools` declaration or a `SKILL.md` that describes what to do without saying which file or directory to do it on." Fix: declare [`allowed-tools`](./skill-invocation-control.md), and make every step reference a concrete artifact or tool.
- **The Wrong Trigger.** A skill fires on adjacent-but-distinct requests — "a skill named `database-migration` keeps firing on SELECT-only queries because the description mentions 'database'." Fix: "tighten the description with the actual trigger conditions ('triggers on schema changes, migration scripts, or CREATE TABLE requests')." Add an explicit "Out of scope" boundary so the skill has negative space.
- **The Drift.** A skill that "worked great for three months, then quietly broke when the underlying tool changed." Fix: "include a version or date in the `SKILL.md` header noting what tool versions you tested against, and add a quarterly review to your repo hygiene."

A fourth, structural one: an **overloaded `SKILL.md`**. Past roughly 5,000 tokens "you've stopped writing a skill and started writing a manual" — split detail into `references/` files the model reads on demand (the same [progressive-disclosure](./skill-content-lifecycle.md) discipline that keeps the body cheap).

## Related concepts

- [Skills](./skills.md) — the core concept; its Troubleshooting table is the reference-side view of these symptoms
- [Skill Frontmatter](./skill-frontmatter.md) — the `description` / `when_to_use` / `allowed-tools` fields this flow reads
- [Skill Invocation Control](./skill-invocation-control.md) — the listing budget the per-prompt scan competes for, and `allowed-tools`
- [Skill Evaluation](./skill-evaluation.md) — measuring and tuning a `description` so it fires on the right prompts
- [Skill Activation Hook](./skill-activation-hook.md) — the deterministic alternative that bypasses model-side matching
- [Description-driven activation vs. the skill-activation hook](../comparisons/description-driven-activation-vs-activation-hook.md) — when to trust the model vs. force the load
- [Path-Scoped Skills](./path-scoped-skills.md) — the prior filter: paths limit visibility, description decides what fires
- [Skill Content Lifecycle](./skill-content-lifecycle.md) — what "load the full body" costs across the rest of the session
