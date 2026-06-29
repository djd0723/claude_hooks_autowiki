---
type: concept
title: "Session Accumulation Pattern (Track-Cheap, Process-Once)"
created: 2026-06-29
updated: 2026-06-29
tags: [hooks, patterns, PostToolUse, Stop, session, incremental, automation]
source_count: 1
sources:
  - sources/clean/github-com-dev-gom-claude-code-marketplace-tree-head-plugins-hook-auto-docs.md
---

# Session Accumulation Pattern (Track-Cheap, Process-Once)

The [hooks adoption ladder](./hooks-adoption-ladder.md) flags the most common trap: "never put `tsc --noEmit` in `PostToolUse` — typecheck is 10–30s; multiply by ~50 edits per feature and you've added ~25 minutes of wall-clock for errors the Stage-4 `Stop` hook catches anyway." Moving the expensive step to `Stop` solves the latency problem, but creates a new one: a na naive `Stop` hook has to process *everything*, even files that didn't change.

The **session accumulation pattern** resolves this: use `PostToolUse` as a lightweight change journal (recording *which* files were touched, not processing them), then use `Stop` to batch-process only the accumulated deltas. The result is constant-time per-edit cost and a single, incremental batch run at the end of each session.

The `hook-auto-docs` plugin from the Claude Code community marketplace demonstrates this pattern concretely: it automatically generates and updates project structure documentation using three coordinated hooks.

## The three-hook choreography

### Stage 0: `SessionStart` — configuration and migration

At session start, the plugin reads its own `plugin.json` to find the current version, then compares against the version stored in `.plugin-config/hook-auto-docs.json`. If they differ (a plugin update happened since the last session), settings are migrated automatically, preserving user customizations while adding new fields with their defaults.

```json
{
  "hooks": {
    "SessionStart": [
      { "hooks": [{ "type": "command", "command": "node scripts/init-config.js" }] }
    ]
  }
}
```

`SessionStart` is the right place for config initialization because it fires even on `--resume` and after `/clear` — the config is always current before any `PostToolUse` or `Stop` hook reads it.

### Stage 1: `PostToolUse` (Write) — lightweight delta journaling

After every `Write` tool call, the plugin records the written file path to `.structure-changes.json`. The hook does nothing else:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write",
        "hooks": [{ "type": "command", "command": "node scripts/track-structure-changes.js" }]
      }
    ]
  }
}
```

This is constant-time regardless of project size. The agent loop pays a small, fixed cost per edit — well under the latency budget — and the journal is a side effect, not a blocking step.

### Stage 2: `Stop` — incremental batch processing

When the session ends, `Stop` runs the expensive operation once, but only on the delta:

1. Read `.structure-changes.json` to get the list of changed files
2. If empty and `.project-structure.md` already exists: no-op (silently skip)
3. If non-empty (or first run): scan only the affected directories, regenerate the structure doc, update `.structure-state.json` with the current file list

The user sees a single message on session end:
```
📚 Auto-Docs: Updated project structure (5 file(s) changed)
```

On the first run, with no prior state, the plugin falls back to a full scan. After that, every subsequent session is incremental.

## The design payoff

| Approach | Per-edit cost | Session-end cost | Accuracy |
|---|---|---|---|
| Expensive hook on every `PostToolUse` | High × N edits | None | Current |
| Naive `Stop`-only (no tracking) | None | High (full scan) | Current |
| Session accumulation | Constant (journal only) | Low (changed files only) | Current |

The session accumulation pattern dominates on large projects where a full scan is expensive but per-edit processing is also too slow. For small projects the difference is negligible; the pattern becomes load-bearing as the project grows.

## Configuration migration via `SessionStart`

A subtle benefit of the `SessionStart` stage: it enables graceful plugin updates. The plugin stores its own version in the config file and compares it against the installed version on every session start. When they differ, migration runs:

- Existing user settings are preserved
- New configuration fields get their defaults
- `_pluginVersion` is bumped so the next start is a no-op

This works because `SessionStart` fires *before* any user interaction — the config is always in a consistent state before `PostToolUse` or `Stop` read it. See [Session Context Injection](./session-context-injection.md) for the full set of session-start matchers (`startup`, `resume`, `clear`, `compact`) that determine when this fires.

## Generalizing the pattern

The hook-auto-docs plugin uses the pattern for documentation, but the structure is reusable for any "process file changes" workflow:

- **Test selection**: journal touched source files in `PostToolUse`, run only their affected tests at `Stop`
- **Cache invalidation**: journal writes, purge only relevant cache entries at `Stop`
- **Audit logging**: journal tool calls cheaply via `async: true`, flush and summarize at `Stop`
- **Dependency analysis**: journal imports when files change, regenerate the affected portion of a dep graph at `Stop`

The constraint that makes the pattern work: the `PostToolUse` step must be cheap enough to run on every tool call without accumulating latency. If the journaling step itself becomes expensive (e.g., parsing the file to extract imports), the latency budget breaks and you're back to the original problem. Journal the *path*, not the *content*.

## Related concepts

- [Per-edit hook processing vs. session-batch processing](../comparisons/per-edit-processing-vs-session-batch.md) — the placement-timing decision this pattern resolves, as a side-by-side
- [Hooks Adoption Ladder](./hooks-adoption-ladder.md) — the source of the "never put tsc in PostToolUse" rule this pattern resolves
- [Hook Automation Use Cases](./hook-automation-use-cases.md) — the broader recipe catalog; this page contributes one concrete pattern
- [Production Hook Patterns (ClaudeKit Case Study)](./production-hook-patterns.md) — sibling case study; ClaudeKit's "stateful threshold" (fire a reminder every N edits) is a lighter version of session-state accumulation
- [Session Context Injection](./session-context-injection.md) — the `SessionStart` matchers and `CLAUDE_ENV_FILE` mechanism used in Stage 0
- [Hook Lifecycle Events](./hook-lifecycle-events.md) — the full event timeline that the three stages attach to
- [Generated Artifact Freshness](./generated-artifact-freshness.md) — a complementary concern: how to *detect* that a generated doc has drifted, as opposed to how to *regenerate* it incrementally
