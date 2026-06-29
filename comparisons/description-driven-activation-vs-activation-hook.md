---
type: comparison
title: "Description-driven activation vs. the skill-activation hook"
created: 2026-06-29
updated: 2026-06-29
tags: [skills, activation, hooks, userpromptsubmit, determinism, intent-classifier, decision]
sources:
  - sources/clean/claudefa-st-blog-guide-mechanics-claude-skills-guide.md
  - sources/clean/claudefa-st-blog-tools-hooks-skill-activation-hook.md
---

# Description-driven activation vs. the skill-activation hook

There are two ways to get a [skill](../concepts/skills.md) to fire on the right prompt, and they
sit at opposite ends of the same axis. The **native, description-driven** path leaves the
decision to the model: on every message Claude semantically matches your prompt against each
skill's `description` and picks what to load. The **[skill-activation hook](../concepts/skill-activation-hook.md)**
path takes that judgment out of the loop: a [`UserPromptSubmit`](../concepts/hook-lifecycle-events.md)
hook keyword-matches your prompt and *appends* skill recommendations before Claude reads the
message. Same goal — the right skill runs — but one trusts the model to remember and the other
guarantees it can't forget. This page maps when each wins and what each costs.

## The fundamental difference: who decides, and when

Native activation is a model decision made *after* Claude sees your prompt:

> "Activation is description-driven, not name-driven."

> "The name alone almost never triggers activation; the description does."

The hook is a deterministic string match made *before* Claude sees your prompt:

> "The Skill Activation Hook intercepts your prompts and appends skill recommendations before
> Claude sees the message. Claude can't forget because it never had to remember."

That clause is the whole axis. Native activation depends on the model *choosing* to load the
right skill from a semantic read of descriptions; the hook turns a soft instruction ("remember
to use this skill") into a hard one (the recommendation is already in the prompt text). This is
the same [deterministic-vs-judgment trade-off](../synthesis.md) that runs through the entire
hooks-and-skills system, applied to the narrow question of *what fires*.

## How each one matches

| | Description-driven (native) | Skill-activation hook |
| :-- | :-- | :-- |
| **Decision-maker** | The model | A `UserPromptSubmit` command hook |
| **Signal** | Semantic match of prompt ↔ `description` | Keyword + regex `intentPatterns` match |
| **Timing** | After the prompt enters context | Before Claude reads the prompt (appended text) |
| **Tuned by** | Writing the `description` as an intent classifier | Editing `keywords`/`intentPatterns` in `skill-rules.json` |
| **Tie-breaking** | "picks the one with the more specific description" | `priority` field groups into CRITICAL vs RECOMMENDED |
| **Failure mode** | Vague description never wins; model forgets | Keywords don't match your speech, or fire too broadly |

The native side treats the `description` as the classifier:

> "Your description field is doing the work of an intent classifier. Write it for that job.
> Mention the trigger phrases users actually type. Mention the file types or task types the
> skill applies to. Skip marketing language."

The hook side moves that same trigger surface out of the description and into config you curate
by hand:

> "If your prompt contains 'commit' or 'git push', the git-commits skill triggers."

with regex to absorb phrasing drift — the pattern `(implement|build).*?feature` catches both
"let's implement this feature" and "build a new feature for me." The maintenance discipline this
implies is explicit and is the hook's real cost: *"After creating any new skill, update its
triggers in skill-rules.json."* Native activation has no such registry — the description *is*
the trigger.

## When to reach for description-driven activation

This is the default and should carry most of the load. Use it when:

- **You want zero added infrastructure.** No hook to install, no `skill-rules.json` to maintain,
  no per-skill trigger registry. You write a good `description` and the model does the rest.
- **The trigger surface is semantic, not lexical.** When the prompts that *should* fire a skill
  don't share obvious keywords — they share *intent* — the model's semantic match beats a
  hand-written keyword list you'd have to keep widening.
- **You're authoring for others.** A shipped skill's `description` travels with it; a keyword
  registry tuned to *your* vocabulary does not.

The price is reliability: native activation depends on the model both having a winning
description *and* choosing to act on it. [Skill evaluation](../concepts/skill-evaluation.md)
exists precisely because "seeing a skill trigger tells you Claude found it, not that it did what
you intended" — and the honest baseline answer is often that it doesn't fire reliably.

## When to reach for the skill-activation hook

The hook earns its maintenance burden when *missing* the skill is expensive and the native path
won't guarantee a hit:

- **A skill must fire every time, not most of the time.** Mark it `critical` and it lands in the
  `CRITICAL SKILLS (REQUIRED)` block of every matching prompt. The model no longer has discretion
  to skip it.
- **Your team's vocabulary is stable and known.** Because matching is driven by your own keyword
  list, the hook adapts to how *you* talk — "If you say 'push my code' rather than 'git push,'
  you add the phrase." A fixed lexicon is cheap to maintain and exact.
- **You want speed and statefulness the model can't give.** The hook "runs in milliseconds," and
  it's stateful — "If it suggested session-management earlier in your conversation, it won't
  repeat the suggestion." Native activation has no memory of what it already recommended.

The standing cautions are the inverse of native's: the hook's three failure modes are *keywords
that don't match your actual speech* (no suggestion), *keywords too broad* (suggestions when not
needed), and *duplicate suggestions* from configuring the hook in both global and project
settings. Every one is a config-tuning problem, not a prompt-writing one.

## They are layers, not rivals

The two compose. Path scoping and discovery decide which skills are *visible*; the description
(or the hook) decides which *fires*. You can run the hook to guarantee a critical skill is
recommended **and** still rely on description-driven activation for the long tail of optional
skills. The hook even depends on the native machinery downstream — its appended `ACTION: Use
Skill tool BEFORE responding` still routes through the ordinary Skill-tool invocation path. The
right reading is a spectrum: write good descriptions for everything, and add the hook only for
the skills you cannot afford to have the model forget.

## Decision table

| Situation | Reach for |
| :-------- | :-------- |
| Default; minimal setup; shipping a skill for others | **Description-driven activation** |
| Trigger is semantic / intent-based, not keyword-shaped | **Description-driven activation** |
| A skill MUST load every time it's relevant | **Skill-activation hook** (`priority: critical`) |
| Team vocabulary is fixed and known | **Skill-activation hook** (keyword list) |
| Need "don't re-suggest what I already loaded" statefulness | **Skill-activation hook** |
| Long tail of optional skills | **Description-driven activation** |
| Both: guarantee the critical few, model-pick the rest | **Run the hook *and* keep good descriptions** |

## Related

- [Skill Activation (Description-Driven)](../concepts/skill-activation.md) — the model-side decision flow and how to write a `description` that fires
- [Skill Activation Hook](../concepts/skill-activation-hook.md) — the deterministic `UserPromptSubmit` pattern and its `skill-rules.json` config
- [Skill Discovery](../concepts/skill-discovery.md) — the prior step: where a skill lives decides whether it's even visible
- [Skill Evaluation](../concepts/skill-evaluation.md) — measuring whether a skill actually invokes when it should
- [Path-Scoped Skills](../concepts/path-scoped-skills.md) — the visibility filter that runs before either activation path
- [CLAUDE.md rules vs. skills workflows](./claude-md-rules-vs-skills-workflows.md) — the prior question: should this even be a skill?
