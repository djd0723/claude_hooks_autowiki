---
type: concept
title: "Skills"
created: 2026-06-29
updated: 2026-06-29
tags: [skills, SKILL.md, extensibility, commands, agent-skills]
source_count: 1
sources:
  - sources/clean/code-claude-com-docs-en-skills-md.md
---

# Skills

A **skill** extends what Claude can do. You create a `SKILL.md` file with instructions, and Claude adds it to its toolkit:

> "Skills extend what Claude can do. Create a `SKILL.md` file with instructions, and Claude adds it to its toolkit. Claude uses skills when relevant, or you can invoke one directly with `/skill-name`."

Two things can trigger a skill: Claude loads it automatically when your request matches its [description](./skill-invocation-control.md), or you invoke it directly by typing `/skill-name`.

Create a skill when you keep pasting the same instructions, checklist, or multi-step procedure into chat, or when a section of CLAUDE.md has grown into a procedure rather than a fact. The decisive advantage over CLAUDE.md:

> "Unlike CLAUDE.md content, a skill's body loads only when it's used, so long reference material costs almost nothing until you need it."

## Skills and custom commands are the same thing

Custom commands have been merged into skills. A file at `.claude/commands/deploy.md` and a skill at `.claude/skills/deploy/SKILL.md` both create `/deploy` and work the same way. Existing `.claude/commands/` files keep working and support the same [frontmatter](./skill-frontmatter.md). Skills add optional features a flat command file can't: a directory for supporting files, frontmatter to [control whether you or Claude invokes them](./skill-invocation-control.md), and automatic loading when relevant. Skills are recommended for new work.

This page covers skills *you author*. For the skills that ship with Claude Code, see [Bundled Skills](./bundled-skills.md).

## The Agent Skills standard

Claude Code skills follow the [Agent Skills](https://agentskills.io) open standard, which works across multiple AI tools. Claude Code extends the standard with additional features:

- [Invocation control](./skill-invocation-control.md) — who can trigger a skill
- [Subagent execution](./skill-subagent-execution.md) — running a skill in a forked context
- [Dynamic context injection](./skill-dynamic-context.md) — inlining live command output before Claude sees the skill

## Anatomy of a skill

Each skill is a **directory** with `SKILL.md` as the required entrypoint. Every `SKILL.md` has two parts: YAML [frontmatter](./skill-frontmatter.md) between `---` markers that tells Claude when to use the skill, and markdown content with the instructions Claude follows when the skill runs. The directory name becomes the command you type.

```text
my-skill/
├── SKILL.md           # Main instructions (required)
├── template.md        # Template for Claude to fill in
├── examples/
│   └── sample.md      # Example output showing expected format
└── scripts/
    └── validate.sh    # Script Claude can execute
```

Only `SKILL.md` is required. Supporting files keep `SKILL.md` focused on essentials while letting Claude load detailed reference material — large reference docs, API specifications, example collections, executable scripts — only when needed:

> "Large reference docs, API specifications, or example collections don't need to load into context every time the skill runs."

Reference supporting files from `SKILL.md` so Claude knows what each contains and when to load it. Keep `SKILL.md` under 500 lines and move detail into separate files.

## Two kinds of skill content

Thinking about how you want to invoke a skill guides what to put in it:

- **Reference content** adds knowledge Claude applies to your current work — conventions, patterns, style guides, domain knowledge. It runs inline so Claude can use it alongside your conversation context.
- **Task content** gives Claude step-by-step instructions for a specific action — deployments, commits, code generation. These are often actions you invoke directly with `/skill-name` rather than letting Claude decide; add [`disable-model-invocation: true`](./skill-invocation-control.md) to keep them manual.

Keep the body concise. Once a skill loads, its content [stays in context across turns](./skill-content-lifecycle.md), so every line is a recurring token cost. State what to do rather than narrating how or why.

## Sharing skills

Skills distribute at different scopes depending on audience (see [Skill Discovery](./skill-discovery.md) for the full precedence rules):

- **Project skills**: commit `.claude/skills/` to version control
- **Plugins**: create a `skills/` directory in your [plugin](./plugin-components.md)
- **Managed**: deploy organization-wide through [managed settings](./settings-files.md)

Add a `.claude-plugin/plugin.json` to a skill folder and it loads as a [skills-directory plugin](./plugin-installation-scopes.md) named `<name>@skills-dir`, so it can bundle agents, hooks, and MCP servers.

## Troubleshooting

| Symptom | Fix |
| :------ | :-- |
| Skill not triggering | Add keywords users would naturally say to the `description`; verify it appears in `What skills are available?`; invoke directly with `/skill-name` |
| Skill triggers too often | Make the `description` more specific, or add `disable-model-invocation: true` |
| Descriptions cut short | Many skills overflow the listing budget; see [Skill Invocation Control](./skill-invocation-control.md) and `/doctor` |
| Malformed frontmatter | Claude Code loads the body with empty metadata, so `/skill-name` works but Claude has no `description` to match. Run `--debug` to see the parse error |

## Related concepts

- [Skill Frontmatter](./skill-frontmatter.md) — every configuration field and how the command name is derived
- [Skill Discovery](./skill-discovery.md) — where skills live, scope/precedence, live change detection
- [Skill Invocation Control](./skill-invocation-control.md) — who can invoke, tool pre-approval, visibility
- [Skill Arguments](./skill-arguments.md) — `$ARGUMENTS`, indexed/named args, string substitutions
- [Skill Dynamic Context](./skill-dynamic-context.md) — `` !`command` `` injection before Claude sees the skill
- [Skill Content Lifecycle](./skill-content-lifecycle.md) — how invoked content persists and survives compaction
- [Skill Subagent Execution](./skill-subagent-execution.md) — `context: fork` and skill ↔ subagent interplay
- [Skill Evaluation](./skill-evaluation.md) — measuring and iterating with `skill-creator`
- [Bundled Skills](./bundled-skills.md) — the skills that ship with Claude Code
