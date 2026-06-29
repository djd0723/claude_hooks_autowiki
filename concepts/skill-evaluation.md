---
type: concept
title: "Skill Evaluation"
created: 2026-06-29
updated: 2026-06-29
tags: [skills, evaluation, skill-creator, testing, baseline]
source_count: 1
sources:
  - sources/clean/code-claude-com-docs-en-skills-md.md
---

# Skill Evaluation

Seeing a [skill](./skills.md) trigger tells you Claude found it, not that it did what you intended. To know a skill works, measure two things separately:

- whether Claude **invokes** it on the prompts it should, and
- whether the **output** matches what you expect when it does.

## The baseline comparison

The check for both is a baseline comparison:

> "Collect a few realistic prompts, run each one in a fresh session with the skill available and again with it disabled, and compare the results."

A fresh session matters because leftover context from authoring the skill will mask gaps in the written instructions. ([Disabling](./skill-invocation-control.md) is done through `skillOverrides`.)

## Automating the loop with `skill-creator`

The `skill-creator` plugin automates the comparison loop inside Claude Code. Install it from the official marketplace:

```text
/plugin install skill-creator@claude-plugins-official
```

If the plugin isn't found, refresh with `/plugin marketplace update claude-plugins-official` (or `/plugin marketplace add anthropics/claude-plugins-official` if not yet added), then retry. After installing, run `/reload-plugins`, then ask Claude to evaluate a skill, e.g. `evaluate my summarize-changes skill with skill-creator`.

What the plugin produces:

| Output | What it does |
| :----- | :----------- |
| Test cases | Stores prompts, input files, and expected behavior in `evals/evals.json` inside the skill directory |
| Isolated runs | Spawns a [subagent](./subagents.md) per test case so each starts clean; records token count and duration |
| Grading | Checks each assertion against the output, writing pass/fail with evidence to `grading.json` |
| Benchmark | Aggregates pass rate, time, and tokens for with-skill vs. without-skill into `benchmark.json` |
| Version comparison | Runs a blind A/B between two skill versions to confirm an edit is an improvement before committing |
| Description tuning | Generates should-trigger and should-not-trigger prompts, measures hit rate, and proposes `description` edits when the skill activates on the wrong requests |
| Review viewer | Opens an HTML report to inspect each output and record qualitative feedback the next iteration reads |

The benchmark lets you weigh the pass-rate improvement against the token and time overhead the skill adds.

## Related concepts

- [Skills](./skills.md) — the core concept
- [Skill Invocation Control](./skill-invocation-control.md) — `skillOverrides`, used to disable a skill for a baseline run
- [Bundled Skills](./bundled-skills.md) — `skill-creator` is distributed as a plugin
