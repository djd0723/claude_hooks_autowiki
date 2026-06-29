---
type: concept
title: "Production Hook Patterns (ClaudeKit Case Study)"
created: 2026-06-29
updated: 2026-06-29
tags: [hooks, patterns, case-study, claudekit, design]
source_count: 1
sources:
  - sources/clean/docs-claudekit-cc-docs-engineer-configuration-hooks.md
---

# Production Hook Patterns (ClaudeKit Case Study)

The [hook lifecycle reference](./hook-lifecycle-events.md) and [hook automation use cases](./hook-automation-use-cases.md) describe what hooks *can* do in the abstract. This page is grounded evidence of what a real, shipping product actually does with them: ClaudeKit Engineer, a third-party kit, ships a curated set of pre-built hooks. Its design choices surface transferable lessons that the abstract references can't.

## The central lesson: context-injection hooks have a recurring cost

A hook that *blocks* or *guards* runs cheaply and only matters when it fires. A hook that *injects context* spends tokens on every event it attaches to — for the whole session. ClaudeKit learned this and changed its defaults accordingly:

> "Current default (v2.19.0+): ClaudeKit installs only lightweight safety and workflow hooks by default. Generated session/subagent/usage context hooks are opt-in and are pruned from existing installs during ck update or ck migrate."

The split is explicit in their built-in catalog: safety/workflow hooks (a simplification gate, file-naming guidance, access blockers) ship **on**; context-generators (session init, subagent context injection, usage-limit awareness, per-prompt rule reminders) ship **off**. The stated rationale:

> "The hooks marked No are no longer installed by default because they generate context on session, prompt, subagent, or tool events. Existing installations are cleaned up idempotently by the CLI during update and migration."

**Takeaway:** default to lightweight guard hooks; treat any hook that adds to the model's context as opt-in, and budget its token cost against the value it adds on *every* invocation.

## Reusable implementation patterns

These are concrete techniques worth borrowing, independent of ClaudeKit:

- **Shared interpreter runner.** Every ClaudeKit hook is invoked as `bash .claude/hooks/node-hook-runner.sh .claude/hooks/<name>.cjs` rather than calling the script directly. A single wrapper centralizes Node resolution, environment setup, and error handling for all hooks — so individual hook files stay focused on logic. (See [Hook Types](./hook-types.md) — these are all `command` handlers.)

- **Sensitive-file gate via a marker + user prompt.** Their `privacy-block.cjs` matches paths like `.env*`, `*.pem`, `*.key`, `credentials.*` on `PreToolUse`, and on a match "returns a `@@PRIVACY_PROMPT@@` JSON marker that triggers the AskUserQuestion flow." If the user approves, "Claude uses bash cat to read the file (bypasses the hook)"; if denied, it proceeds without the file. This is a decision-control pattern (see [Hook Decision Control](./hook-decision-control.md)) that escalates to the human instead of silently allowing or denying. Compare [File Permission Patterns](./file-permission-patterns.md).

- **Layered ignore-list blocking.** Their `scout-block.cjs` blocks `Read/Edit/Write/Glob/Grep` against an ignore list, "reads the shipped `.ckignore` baseline and then layers an optional git-root `./.claude/.ckignore` override," and still "allows Bash commands that are recognized build/test commands." A shipped baseline plus a project override is a clean way to make a guard hook both safe-by-default and locally tunable.

- **Stateful threshold reminders.** Their `post-edit-simplify-reminder.cjs` increments an edit counter in session state on each `Edit/Write/MultiEdit` and only injects a reminder once a threshold (default 5) is crossed, then resets. Hooks can hold session state to fire *periodically* rather than on every event — keeping a noisy reminder useful.

- **Deliberately manual notifications.** ClaudeKit's Discord notifier is *not* wired to a hook event:

> "Discord notifications are triggered manually in workflows, not automatically via hook events. This is intentional for flexibility."

  Not every automation belongs on an event. A side-effect with cost or noise (an external webhook) can be better invoked explicitly from a workflow than fired on every `Stop`.

## A note on event names

ClaudeKit's docs list events including `SubagentStart`, `TaskCompleted`, and `TeammateIdle` for its Agent Team workflows. These align with the broader event set in the [hook lifecycle reference](./hook-lifecycle-events.md); ClaudeKit exposes a curated subset relevant to its kit rather than the full Claude Code surface.

## Related concepts

- [Hook Automation Use Cases](./hook-automation-use-cases.md) — generic recipes; this page is the real-world counterpart
- [Hook Lifecycle Events](./hook-lifecycle-events.md) — the events these hooks attach to
- [Hook Types](./hook-types.md) — the `command` handler every ClaudeKit hook uses
- [Hook Decision Control](./hook-decision-control.md) — the block/allow/escalate mechanics behind the privacy and access gates
- [Hook Scope](./hook-scope.md) — these are all project-scoped (`.claude/settings.json`) hooks
- [File Permission Patterns](./file-permission-patterns.md) — the access-control model the ignore-list and privacy gates complement
- [AI Permission Reviewer (ClaudeFast Permission Hook Case Study)](./ai-permission-reviewer.md) — a sibling third-party case study, focused on the `PermissionRequest` permission surface
