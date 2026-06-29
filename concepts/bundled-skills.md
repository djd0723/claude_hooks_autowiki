---
type: concept
title: "Bundled Skills"
created: 2026-06-29
updated: 2026-06-29
tags: [skills, bundled, run, verify, commands]
source_count: 1
sources:
  - sources/clean/code-claude-com-docs-en-skills-md.md
---

# Bundled Skills

Claude Code ships with a set of bundled [skills](./skills.md) available in every session, including `/code-review`, `/batch`, `/debug`, `/loop`, and `/claude-api`. They are disabled only by the `disableBundledSkills` setting.

What distinguishes them from most built-in commands:

> "Unlike most built-in commands, which execute fixed logic directly, bundled skills are prompt-based: they give Claude detailed instructions and let it orchestrate the work using its tools."

You invoke them the same way as any other skill, by typing `/` followed by the name. They are listed alongside built-in commands in the commands reference, marked **Skill** in the Purpose column.

A bundled skill can be overridden: a skill of the same name at any [discovery level](./skill-discovery.md) (e.g. a project `.claude/skills/code-review/`) replaces the bundled one.

## Run and verify your app

Three bundled skills work together to launch your app and confirm changes against the *running* app instead of just tests:

| Skill | Purpose |
| :---- | :------ |
| `/run` | Launch and drive your app to see a change working |
| `/verify` | Build and run your app to confirm a code change does what it should, without falling back to tests or type checks |
| `/run-skill-generator` | Teach `/run` and `/verify` how to build and launch your project |

All three require Claude Code v2.1.145 or later.

`/run` and `/verify` work without setup — they infer the launch from your project type (CLI, server, TUI, browser-driven) and from your README, `package.json`, or `Makefile`. That inference gets unreliable for projects needing more than a standard launch: a database, an env file, a graphical session, a multi-step build.

`/run-skill-generator` records the recipe instead. It gets your app running from a clean environment, captures what worked (install commands, env vars, launch script), and commits it as a per-project skill at `.claude/skills/run-<name>/`. After that, `/run`, `/verify`, and any other agent in the repo follow the recorded recipe instead of rediscovering it. Run it once per project, and again if the build or launch process changes.

## Related concepts

- [Skills](./skills.md) — authoring your own skills
- [Skill Discovery](./skill-discovery.md) — how a same-named skill overrides a bundled one
- [Skill Evaluation](./skill-evaluation.md) — the `skill-creator` plugin for measuring skills
