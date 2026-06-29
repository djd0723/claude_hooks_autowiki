---
type: summary
title: "Skills — authoring, discovery, and invocation"
slug: code-claude-com-docs-en-skills-md
created: 2026-06-29
updated: 2026-06-29
tags: [skills, authoring, discovery, invocation, frontmatter, progressive-disclosure, subagents]
sources:
  - sources/clean/code-claude-com-docs-en-skills-md.md
---

# Summary: Extend Claude with skills

The authoritative reference for creating, configuring, discovering, and invoking Claude Code skills: `SKILL.md` authoring, frontmatter fields, who-invokes control, the content lifecycle, dynamic context injection, subagent execution, evaluation, and sharing.

## Key thesis from source

> "Skills extend what Claude can do. Create a `SKILL.md` file with instructions, and Claude adds it to its toolkit. Claude uses skills when relevant, or you can invoke one directly with `/skill-name`."

The source frames when to reach for a skill rather than CLAUDE.md:

> "Create a skill when you keep pasting the same instructions, checklist, or multi-step procedure into chat, or when a section of CLAUDE.md has grown into a procedure rather than a fact. Unlike CLAUDE.md content, a skill's body loads only when it's used, so long reference material costs almost nothing until you need it."

## What this source covers

- What a [skill](../concepts/skills.md) is and how custom commands merged into skills
- The [`bundled skills`](../concepts/bundled-skills.md) set and the run/verify trio
- [`SKILL.md` frontmatter](../concepts/skill-frontmatter.md) reference and command-name derivation
- [Progressive disclosure / content lifecycle](../concepts/skill-content-lifecycle.md): descriptions vs full body, auto-compaction budget
- [Discovery](../concepts/skill-discovery.md) across locations, nested directories, and `--add-dir`
- [Invocation control](../concepts/skill-invocation-control.md): who can invoke and tool pre-approval
- [Arguments](../concepts/skill-arguments.md) and string substitutions
- [Dynamic context injection](../concepts/skill-dynamic-context.md) with `` !`<command>` ``
- [Subagent execution](../concepts/skill-subagent-execution.md) via `context: fork`
- [Evaluation](../concepts/skill-evaluation.md) with baseline comparison and `skill-creator`

## What a skill is

A skill is a directory whose `SKILL.md` entrypoint holds YAML frontmatter (telling Claude when to use it) and markdown instructions (what Claude follows when it runs). The directory name becomes the `/` command. Custom commands were merged into skills:

> "**Custom commands have been merged into skills.** A file at `.claude/commands/deploy.md` and a skill at `.claude/skills/deploy/SKILL.md` both create `/deploy` and work the same way."

Skills follow the [Agent Skills](https://agentskills.io) open standard; Claude Code extends it with invocation control, subagent execution, and dynamic context injection.

## Where skills live

Location determines who can use a skill:

| Location | Path | Applies to |
| :------- | :--- | :--------- |
| Enterprise | managed settings | All users in your organization |
| Personal | `~/.claude/skills/<skill-name>/SKILL.md` | All your projects |
| Project | `.claude/skills/<skill-name>/SKILL.md` | This project only |
| Plugin | `<plugin>/skills/<skill-name>/SKILL.md` | Where plugin is enabled |

Override precedence for same-named skills: **enterprise > personal > project**, and any of these overrides a bundled skill. Plugin skills use a `plugin-name:skill-name` namespace and cannot conflict. A skill takes precedence over a same-named `.claude/commands/` file.

Skills also load from nested `.claude/skills/` directories on demand (monorepo support), and from `.claude/skills/` inside `--add-dir`/`/add-dir` directories — an explicit exception to the rule that additional directories grant file access only, not configuration discovery. `permissions.additionalDirectories` does **not** load skills.

## Bundled skills

> "Claude Code includes a set of bundled skills that are available in every session unless disabled with the [`disableBundledSkills`](/en/settings#available-settings) setting, including `/code-review`, `/batch`, `/debug`, `/loop`, and `/claude-api`."

Unlike most built-in commands (fixed logic), bundled skills are prompt-based: they instruct Claude and let it orchestrate using its tools. Three work together to run and verify your app (all require v2.1.145+):

| Skill | Purpose |
| :---- | :------ |
| `/run` | Launch and drive your app to see a change working |
| `/verify` | Build and run your app to confirm a change does what it should, without falling back to tests or type checks |
| `/run-skill-generator` | Teach `/run` and `/verify` how to build and launch your project |

## Frontmatter reference

All fields are optional; only `description` is recommended. Key fields:

| Field | Required | Description |
| :---- | :------- | :---------- |
| `name` | No | Display name in listings. Defaults to directory name; does not change the command typed (except plugin-root `SKILL.md`) |
| `description` | Recommended | What the skill does and when to use it; Claude matches on this. Combined with `when_to_use`, truncated at 1,536 chars in the listing |
| `when_to_use` | No | Extra trigger context; appended to `description`, counts toward the 1,536-char cap |
| `argument-hint` | No | Autocomplete hint, e.g. `[issue-number]` |
| `arguments` | No | Named positional arguments for `$name` substitution |
| `disable-model-invocation` | No | `true` blocks automatic loading (manual `/name` only); also blocks subagent preloading. Default `false` |
| `user-invocable` | No | `false` hides from the `/` menu (Claude-only). Default `true` |
| `allowed-tools` | No | Tools usable without permission prompt while the skill is active |
| `disallowed-tools` | No | Tools removed from the pool while active; clears on next message |
| `model` | No | Model override for the rest of the turn; not saved to settings |
| `effort` | No | Effort level override: `low`/`medium`/`high`/`xhigh`/`max` |
| `context` | No | `fork` runs the skill in a forked subagent context |
| `agent` | No | Subagent type to use when `context: fork` is set |
| `hooks` | No | Hooks scoped to this skill's lifecycle |
| `paths` | No | Glob patterns that limit when the skill auto-activates |
| `shell` | No | `bash` (default) or `powershell` for inline shell commands |

### How a skill gets its command name

The typed command comes from where the file lives, not from `name`:

| Skill location | Command name source | Example |
| :------------- | :------------------ | :------ |
| `~/.claude/skills/` or `.claude/skills/` | Directory name | `.claude/skills/deploy-staging/` → `/deploy-staging` |
| Nested dir on name clash | Subdir path + skill dir name | `apps/web/.claude/skills/deploy/` → `/apps/web:deploy` |
| `.claude/commands/` file | File name without extension | `deploy.md` → `/deploy` |
| Plugin `skills/` subdir | Directory name, plugin-namespaced | `my-plugin/skills/review/` → `/my-plugin:review` |
| Plugin root `SKILL.md` | Frontmatter `name`, plugin dir as fallback | `name: review` → `/my-plugin:review` |

The plugin-root case is the one place `name` sets the command name.

## Skill content lifecycle

> "When you or Claude invoke a skill, the rendered `SKILL.md` content enters the conversation as a single message and stays there for the rest of the session. Claude Code does not re-read the skill file on later turns, so write guidance that should apply throughout a task as standing instructions rather than one-time steps."

Two content styles guide what to include:

- **Reference content** — conventions, patterns, domain knowledge applied inline alongside conversation context.
- **Task content** — step-by-step actions (deploy, commit) often paired with `disable-model-invocation: true`.

Keep the body concise; every line is a recurring token cost. Auto-compaction carries invoked skills forward within a budget: it re-attaches the most recent invocation of each skill after the summary, keeping the first 5,000 tokens of each, sharing a combined **25,000-token** budget filled from the most recently invoked, so older skills can be dropped.

## Progressive disclosure: descriptions vs full body

In a regular session, descriptions are always loaded so Claude knows what's available; full content loads only on invocation. The two control fields shape this:

| Frontmatter | You can invoke | Claude can invoke | When loaded into context |
| :---------- | :------------- | :---------------- | :----------------------- |
| (default) | Yes | Yes | Description always in context, full skill loads when invoked |
| `disable-model-invocation: true` | Yes | No | Description not in context, full skill loads when you invoke |
| `user-invocable: false` | No | Yes | Description always in context, full skill loads when invoked |

Subagents with preloaded skills differ: the full content is injected at startup. Supporting files (templates, reference docs, scripts) stay out of context until referenced — keep `SKILL.md` under 500 lines.

## Pre-approve tools

> "The `allowed-tools` field grants permission for the listed tools while the skill is active, so Claude can use them without prompting you for approval. It does not restrict which tools are available."

For project `.claude/skills/`, `allowed-tools` takes effect only after accepting the workspace trust dialog — review project skills before trusting a repo, since a skill can grant itself broad access. `disallowed-tools` removes tools from the pool; the restriction clears on your next message.

## Arguments and substitutions

Both you and Claude can pass arguments. Available substitutions:

| Variable | Description |
| :------- | :---------- |
| `$ARGUMENTS` | All arguments as typed. If absent, arguments are appended as `ARGUMENTS: <value>` |
| `$ARGUMENTS[N]` | Argument by 0-based index |
| `$N` | Shorthand for `$ARGUMENTS[N]` (`$0` is first) |
| `$name` | Named argument from the `arguments` frontmatter list, mapped by position |
| `${CLAUDE_SESSION_ID}` | Current session ID |
| `${CLAUDE_EFFORT}` | Current effort level (ultracode reports as `xhigh`) |
| `${CLAUDE_SKILL_DIR}` | Directory containing the skill's `SKILL.md` |

Indexed arguments use shell-style quoting (wrap multi-word values in quotes). Escape a literal `$` before a digit/`ARGUMENTS`/declared name with a backslash (`\$1.00`).

## Inject dynamic context

> "The `` !`<command>` `` syntax runs shell commands before the skill content is sent to Claude. The command output replaces the placeholder, so Claude receives actual data, not the command itself."

This is preprocessing, run once over the original file: output is inserted as plain text and not re-scanned for further placeholders. The inline form is recognized only when `!` starts a line or follows whitespace (`KEY=!`cmd`` stays literal). Use a ` ```! ` fenced block for multi-line commands. `"disableSkillShellExecution": true` in settings replaces each command with `[shell command execution disabled by policy]`; bundled and managed skills are exempt.

## Run skills in a subagent

`context: fork` runs the skill in isolation: the skill content becomes the subagent's prompt, with no access to conversation history. It only makes sense for skills with explicit task instructions, not bare guidelines. Skills and subagents interact in two directions:

| Approach | System prompt | Task | Also loads |
| :------- | :------------ | :--- | :--------- |
| Skill with `context: fork` | From agent type | SKILL.md content | CLAUDE.md, except when the agent is Explore or Plan |
| Subagent with `skills` field | Subagent's markdown body | Claude's delegation message | Preloaded skills + CLAUDE.md |

The `agent` field picks the execution environment (`Explore`, `Plan`, `general-purpose`, or any custom agent); defaults to `general-purpose`.

## Restrict Claude's skill access

Three ways to control which skills Claude can invoke:

- **Disable all** — deny the `Skill` tool in `/permissions`.
- **Allow/deny specific** — permission rules: `Skill(name)` (exact), `Skill(name *)` (prefix with any args).
- **Hide individual** — `disable-model-invocation: true` removes the skill from Claude's context.

`user-invocable: false` controls only `/` menu visibility, not Skill tool access. A few built-in commands (`/init`, `/review`, `/security-review`) are reachable through the Skill tool; others like `/compact` are not.

### Override visibility from settings

`skillOverrides` controls visibility without editing the skill's frontmatter (the `/skills` menu writes it to `.claude/settings.local.json`):

| Value | Listed to Claude | In `/` menu |
| :---- | :--------------- | :---------- |
| `"on"` | Name and description | Yes |
| `"name-only"` | Name only | Yes |
| `"user-invocable-only"` | Hidden | Yes |
| `"off"` | Hidden | Hidden |

A skill absent from `skillOverrides` is treated as `"on"`. Plugin skills are unaffected — manage them via `/plugin`.

## Evaluate and iterate

> "Seeing a skill trigger tells you Claude found it, not that it did what you intended. To know a skill is working, measure two things separately: whether Claude invokes it on the prompts it should, and whether the output matches what you expect when it does."

The check for both is a baseline comparison: run realistic prompts in a fresh session with the skill available and again with it disabled, and compare. The `skill-creator` plugin automates the loop — test cases (`evals/evals.json`), isolated subagent runs, grading (`grading.json`), benchmark aggregation (`benchmark.json`), blind A/B version comparison, description tuning, and an HTML review viewer.

## Troubleshooting highlights

- **Not triggering**: add natural keywords to `description`; verify via `What skills are available?`; rephrase; invoke directly. Malformed frontmatter YAML loads the body with empty metadata (so `/name` works but Claude can't match) — use `--debug`.
- **Triggers too often**: make the description more specific, or add `disable-model-invocation: true`.
- **Descriptions cut short**: the listing budget scales at 1% of context window; least-used descriptions drop first. Run `/doctor` to see shortening/drops. Raise with `skillListingBudgetFraction` or `SLASH_COMMAND_TOOL_CHAR_BUDGET`; per-entry cap is `maxSkillDescriptionChars` (default 1,536).

## Relationship to concept pages

This source is the primary reference for:
- [Skills](../concepts/skills.md) — what a skill is, locations, custom-command merge
- [Skill frontmatter](../concepts/skill-frontmatter.md) — full field reference and command-name derivation
- [Skill discovery](../concepts/skill-discovery.md) — locations, nested/parent dirs, `--add-dir`, live change detection
- [Skill content lifecycle](../concepts/skill-content-lifecycle.md) — single-message injection, compaction budget
- [Skill dynamic context](../concepts/skill-dynamic-context.md) — `` !`<command>` `` injection and policy controls
- [Skill arguments](../concepts/skill-arguments.md) — `$ARGUMENTS`, indexed/named args, substitutions
- [Skill invocation control](../concepts/skill-invocation-control.md) — who-invokes fields, permissions, `skillOverrides`
- [Skill subagent execution](../concepts/skill-subagent-execution.md) — `context: fork` and the agent field
- [Skill evaluation](../concepts/skill-evaluation.md) — baseline comparison and `skill-creator`
- [Bundled skills](../concepts/bundled-skills.md) — the always-available prompt-based skill set
- [SDK skills loading](../concepts/sdk-skills-loading.md) — how skills load outside interactive sessions
