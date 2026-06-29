---
type: concept
title: "Skill Activation Hook (Deterministic Skill Loading Pattern)"
created: 2026-06-29
updated: 2026-06-29
tags: [hooks, skills, userpromptsubmit, skill-rules, determinism, prompt-injection]
source_count: 1
sources:
  - sources/clean/claudefa-st-blog-tools-hooks-skill-activation-hook.md
---

# Skill Activation Hook (Deterministic Skill Loading Pattern)

[Skill evaluation](./skill-evaluation.md) measures whether Claude *invokes* a skill when it should — and the honest answer is often "not reliably." Telling Claude to load a skill, or documenting it in `CLAUDE.md`, both depend on the model *remembering* to act. This page captures one practitioner pattern that removes memory from the loop entirely: a [`UserPromptSubmit`](./hook-lifecycle-events.md) hook that appends skill recommendations to every prompt before Claude reads it.

> "The Skill Activation Hook intercepts your prompts and appends skill recommendations before Claude sees the message. Claude can't forget because it never had to remember."

That last clause is the whole idea — turning a soft instruction ("remember to use this skill") into a hard one (the recommendation is already in the prompt).

## What Claude actually receives

When you type `help me implement a feature`, the hook rewrites what reaches the model:

```
help me implement a feature

SKILL ACTIVATION CHECK

CRITICAL SKILLS (REQUIRED):
  -> session-management

RECOMMENDED SKILLS:
  -> git-commits

ACTION: Use Skill tool BEFORE responding
```

The recommendation block is appended text, fired on the `UserPromptSubmit` event "before Claude sees anything." This is the same injection point the [hook I/O reference](./hook-input-output.md) describes for `additionalContext` — context added *"Alongside the submitted prompt."* The hook "runs in milliseconds," so there's no perceptible delay.

## The matching system

The hook decides *which* skills to recommend by matching the prompt two ways, working together:

- **Keyword matching** — plain string matching. "If your prompt contains 'commit' or 'git push', the git-commits skill triggers."
- **Intent patterns** — regex for natural-language variation. The pattern `(implement|build).*?feature` catches both "let's implement this feature" and "build a new feature for me."

Keywords are cheap and exact; intent patterns absorb phrasing drift. Using both means a skill fires on the obvious word *and* on paraphrases of the same intent.

## Configuration: `skill-rules.json`

Triggers live per-skill in `.claude/skills/skill-rules.json`:

```json
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

`priority` controls how recommendations are grouped in the appended block:

| Priority | Meaning |
| :------- | :------ |
| Critical | "Must load before any work" |
| High | "Strongly recommended" |
| Medium | "Helpful context" |
| Low | "Optional enhancement" |

Critical skills land in the `CRITICAL SKILLS (REQUIRED)` group; lower priorities fall under `RECOMMENDED SKILLS`.

## Tuning to how *you* talk

Because matching is driven by your own keyword list, the hook adapts to your vocabulary. If you say "push my code" rather than "git push," you add the phrase:

```json
"keywords": ["commit", "git push", "push my code", "commit changes"]
```

The maintenance discipline this implies: *every new skill needs its triggers registered*. As the source puts it — "After creating any new skill, update its triggers in skill-rules.json. Then read those triggers and keep those speech patterns in mind when prompting." This is the inverse of the [model-side, description-driven activation](./skill-activation.md) that [skill discovery](./skill-discovery.md) and [invocation control](./skill-invocation-control.md) describe: here *you* curate the trigger surface deterministically rather than relying on the model to weigh skill descriptions.

## Session intelligence (no nagging)

The hook is stateful so it doesn't repeat itself:

> "If it suggested session-management earlier in your conversation, it won't repeat the suggestion. Less noise, same coverage."

State lives in `recommendation-log.json` and "auto-cleans after 7 days." This is the same family of [stateful-threshold technique seen in other production hooks](./production-hook-patterns.md) — holding session state so an automation fires meaningfully rather than on every single event.

## Wiring and testing

It's a `UserPromptSubmit` command hook in `.claude/settings.local.json` — a `.cmd` wrapper on Windows, a `bash …skill-activation-prompt.sh` on Linux/Mac. Because it's an ordinary [command hook](./hook-types.md), you can exercise it by piping a fake event to the script:

```bash
echo '{"session_id":"test","prompt":"implement a feature"}' | node .claude/hooks/SkillActivationHook/skill-activation-prompt.mjs
```

Three failure modes worth knowing:

- **No suggestions appearing** — your keywords don't match your actual speech; test manually and widen them.
- **Suggestions when not needed** — keywords too broad; tighten to more specific terms or intent patterns.
- **Duplicate suggestions** — the hook is configured in *both* global and project settings; keep it in one location only.

## Why it matters

The pattern's payoff is stated plainly: "The Skill Activation Hook removes human memory from the equation." It trades the model's unreliable recall for a deterministic, keyword-driven guarantee — the same philosophy behind treating hooks, not prompts, as the place to *enforce* behavior. It is a sibling to the other claudefa.st practitioner patterns ([setup hooks](./setup-hooks.md), the [AI permission reviewer](./ai-permission-reviewer.md)): each pushes a responsibility Claude would otherwise have to *remember* down into a deterministic hook.

## Related concepts

- [Hook Lifecycle Events](./hook-lifecycle-events.md) — the `UserPromptSubmit` event this hook attaches to
- [Hook Input/Output](./hook-input-output.md) — `additionalContext` / prompt-adjacent injection, the mechanism behind the appended block
- [Hook Types](./hook-types.md) — the `command` handler this hook uses
- [Skill Activation (Description-Driven)](./skill-activation.md) — the model-side matching this hook bypasses
- [Skill Evaluation](./skill-evaluation.md) — measuring the invocation-reliability problem this hook solves deterministically
- [Skill Discovery](./skill-discovery.md) — the model-driven side of which skills are available
- [Skill Invocation Control](./skill-invocation-control.md) — `skillOverrides` and per-skill visibility, the config-side complement
- [Production Hook Patterns](./production-hook-patterns.md) — sibling real-world hooks, incl. the stateful-reminder technique
- [Setup Hooks](./setup-hooks.md) — a sibling claudefa.st practitioner pattern
- [Description-driven activation vs. the skill-activation hook](../comparisons/description-driven-activation-vs-activation-hook.md) — this hook weighed against the native model-side path
