---
type: concept
title: "Path-Scoped Skills"
created: 2026-06-29
updated: 2026-06-29
tags: [skills, paths, discovery, monorepo, claude-md, progressive-disclosure, practitioner-pattern]
source_count: 1
sources:
  - sources/clean/claudefa-st-blog-guide-mechanics-path-scoped-skills.md
---

# Path-Scoped Skills

The [`paths:` frontmatter field](./skill-frontmatter.md) binds a [skill](./skills.md) to a directory glob so it only enters [discovery](./skill-discovery.md) when Claude touches a matching path. The motivating problem:

> "You have 20 skills installed. Claude is working in services/billing. Eighteen of those skills have nothing to do with billing, but their descriptions still get listed at session start, eating tokens you do not get back. That is the problem path-scoped skills in Claude Code were designed to solve: bind a skill to a directory glob so Claude only sees it when the work touches that path."

This is the *practitioner's* view of the `paths:` field that [Skill Frontmatter](./skill-frontmatter.md) lists in its reference table: when to reach for it, how to scope it, and the failure modes.

## Rules vs. Workflows: the mental model

Path scoping forces a prior question — should this knowledge be a skill at all, or a [CLAUDE.md](./sdk-project-instructions.md) rule? The source calls the wrong answer "the single most common skill mistake."

> "CLAUDE.md holds rules. Conventions that must hold every time Claude touches the relevant area. Loaded automatically, always in context."

> "Skills hold workflows. Multi-step procedures that fire only on specific task types. Loaded on demand."

The decision tree (both can be path-scoped — the question is which you reach for):

- **Every-time convention** → it's a rule. Put it in a layered CLAUDE.md.
- **Specific-task procedure** (adding a route, reviewing money code, writing a migration) → it's a workflow. Put it in a skill.
- **Both** → split it. The convention goes in CLAUDE.md, the procedure goes in a skill, and the skill references the CLAUDE.md rule.

Get it wrong and "you pay twice":

> "Rules in skills means Claude misses the rule on the 80% of tasks where the skill never activates. Workflows in CLAUDE.md means the procedure burns tokens in every session even when irrelevant, the bloat problem the skill listing budget was added to police."

## How the `paths:` filter works

The key lives in the YAML frontmatter of any `SKILL.md`. It accepts a single glob string or a YAML list of globs:

```yaml
---
name: billing-money-rules
description: >-
  Use when creating or editing code under services/billing. Enforces money rules:
  integer cents only, seat limits, and BillingError for every violation.
paths: services/billing/**
---
```

```yaml
paths:
  - services/**
  - packages/**
  - tests/**
```

> "Globs follow standard rules: ** matches any nesting depth, * matches one segment, prefix paths are relative to the project root."

The filter runs *before* descriptions reach the model:

> "it scans the file paths Claude has touched or is about to touch. Skills whose paths: glob matches enter the discovery set. Non-matching skills get filtered out before their descriptions land in the system prompt."

That makes path scoping a deeper level of [progressive disclosure](./skill-content-lifecycle.md):

> "not just 'load the body when needed' but 'do not list the skill if the work is somewhere else.'"

## Path scoping is one filter of two

Scoping shrinks the candidate set; it does not replace the description. [Discovery is description-driven](./skill-discovery.md), so the surviving skills still compete on their `description:`:

> "Scoping does not let you write lazy descriptions. The path filter shrinks the candidate set; the description still has to win against the others in that set. 'Helps with backend code' loses to 'Use when adding an HTTP route under services/api' every time."

The [skill activation hook](./skill-activation-hook.md) is the deterministic backstop:

> "Pair it with path scoping and you get a two-stage filter: paths limit visibility, description plus hook decide what fires."

## Scoping strategies that work

> "Four patterns cover almost every real case."

- **Service-specific skills.** One skill per top-level service, scoped to that service root — `billing-money-rules` at `services/billing/**`, `api-add-route` at `services/api/**`. "Maps cleanly onto microservice or workspace boundaries."
- **File-type skills.** Scope by extension or filename when the workflow is tied to a file type — a migration-review skill at `db/migrations/**/*.sql`, a translations skill at `**/i18n/*.json`.
- **Mirror your folder structure.** "One per package, one per service, named after its scope. New engineers can guess where a skill lives before opening the folder."
- **Do not scope cross-cutting skills.** A git-commit skill, a code-review checklist, an incident-response runbook fire regardless of location. "Adding paths: to these is a categorization error: they are not bound to a location."

## Composing with layered CLAUDE.md

Path-scoped skills sit alongside [subdirectory CLAUDE.md](./sdk-project-instructions.md) files, so each folder gets its own harness:

```
project/
├── CLAUDE.md                              # cross-cutting conventions
├── services/
│   ├── api/CLAUDE.md                      # API rules
│   └── billing/CLAUDE.md                  # billing rules
└── .claude/skills/
    ├── api-add-route/SKILL.md             # paths: services/api/**
    ├── billing-money-rules/SKILL.md       # paths: services/billing/**
    └── scoped-tests/SKILL.md              # paths: services/**, packages/**
```

> "Inside services/api, Claude loads the root CLAUDE.md, the services/api/CLAUDE.md, and the api-add-route skill enters discovery. Inside services/billing, Claude sees the billing rules, the billing CLAUDE.md, and a different skill. Same Claude, two different harnesses, zero overlap."

The folder-driven half of this — root-and-nested skill discovery in a monorepo — is detailed in [Skill Discovery](./skill-discovery.md).

## Anti-patterns

> "Three failure modes show up consistently."

- **Scoping too narrowly.** A glob like `services/api/handlers/payments.py` only fires when Claude edits that exact file. "Scope to the directory, not the file."
- **Scoping too broadly.** "paths: **/* is the same as not scoping. It surfaces the skill on every session, which is the bloat you were trying to avoid."
- **Mixing rules into skills** (or workflows into CLAUDE.md). "If you find yourself writing 'always do X' in a SKILL.md, it belongs in CLAUDE.md. If you find yourself writing a 12-step procedure in CLAUDE.md, it belongs in a skill."

Path scoping is also the cleanest lever for staying under the [skill listing budget](./skill-invocation-control.md):

> "The skill listing budget setting (skillListingBudgetFraction) silently drops descriptions when the listing exceeds 1% of context. Path scoping is one of the cleanest ways to stay under that budget without losing skills."

## Related concepts

- [Skills](./skills.md) — what a skill is
- [Skill Frontmatter](./skill-frontmatter.md) — the `paths:` field in the full frontmatter reference
- [Skill Discovery](./skill-discovery.md) — the folder-driven side: nested/monorepo discovery and command-name derivation
- [Skill Invocation Control](./skill-invocation-control.md) — the `skillListingBudgetFraction` budget this scoping protects
- [Skill Content Lifecycle](./skill-content-lifecycle.md) — progressive disclosure, the principle path scoping extends
- [Skill Activation Hook](./skill-activation-hook.md) — the deterministic second stage of the two-stage filter
- [SDK Project Instructions](./sdk-project-instructions.md) — CLAUDE.md, the home of rules in the rules-vs-workflows split
- [CLAUDE.md rules vs. skills workflows](../comparisons/claude-md-rules-vs-skills-workflows.md) — the rules-vs-workflows decision this page surfaces, expanded
- [Large-Codebase Playbook](./large-codebase-playbook.md) — path-scoped skills as Strategy 8 of scaling the harness
