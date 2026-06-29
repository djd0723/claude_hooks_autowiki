---
type: comparison
title: "Per-edit hook processing vs. session-batch processing"
created: 2026-06-29
updated: 2026-06-29
tags: [hooks, PostToolUse, Stop, latency, batch, incremental, patterns]
sources:
  - sources/clean/github-com-dev-gom-claude-code-marketplace-tree-head-plugins-hook-auto-docs.md
  - sources/clean/thomas-wiegold-com-blog-claude-code-hooks.md
---

# Per-edit hook processing vs. session-batch processing

When an automation needs to react to file changes — regenerate docs, run typecheck, select tests, invalidate caches — there are two places to put the expensive work: run it on every `PostToolUse` edit, or defer it to a single `Stop` hook at the end of the session. This is a *placement-timing* decision, orthogonal to [sync vs async execution](./sync-vs-async-hooks.md) (which is about whether Claude blocks on the hook). Getting it wrong is the single most common hooks anti-pattern.

## The anti-pattern that forces the choice

The [hooks adoption ladder](../concepts/hooks-adoption-ladder.md) names the trap directly: putting an expensive step in `PostToolUse` multiplies its cost by every edit.

> "never put `tsc --noEmit` in `PostToolUse` — typecheck is 10–30s; multiply by ~50 edits per feature and you've added ~25 minutes of wall-clock for errors the Stage-4 `Stop` hook catches anyway."

Moving the expensive step to `Stop` fixes the latency, but a naive `Stop` hook then has to reprocess *everything* — even files that never changed. The [session accumulation pattern](../concepts/session-accumulation-pattern.md) resolves the dilemma by splitting the work across both events.

## Comparison

| Dimension | Per-edit (`PostToolUse` does the work) | Naive session-batch (`Stop` only) | Session accumulation (journal + batch) |
| :-------- | :------------------------------------- | :-------------------------------- | :------------------------------------- |
| Per-edit cost | High × N edits | None | Constant (journal the path only) |
| Session-end cost | None | High (full scan) | Low (changed files only) |
| Latency on the agent loop | Accumulates badly | None | Negligible |
| Scope processed | Just the touched file | Whole project | Accumulated delta |
| Where the cost lands | Hot path (Claude waits) | One blocking step at end | Mostly off the hot path |
| Best when | Work is genuinely cheap (format/lint one file) | Project is small; full scan is fast | Large project; per-edit too slow AND full scan too slow |

## When to use which

**Per-edit `PostToolUse`** — correct only when the operation is *intrinsically* cheap and scoped to the single touched file: formatting, autofix-lint, a one-file schema validation. The adoption ladder's Stage 1 ("format & lint on every edit") is the canonical fit.

**Naive `Stop`-only** — fine for small projects where a full scan is fast enough that tracking deltas isn't worth the complexity. Process everything once at session end.

**Session accumulation** — the load-bearing choice for large projects: `PostToolUse` writes the changed path to a journal file (constant-time, off the hot path), and `Stop` reads the journal and processes only the delta.

> "The constraint that makes the pattern work: the `PostToolUse` step must be cheap enough to run on every tool call without accumulating latency. ... Journal the *path*, not the *content*."

## Key constraint

The split only pays off if the journaling step stays trivial. The moment the `PostToolUse` step does real work (parsing the file, resolving imports), the latency budget breaks and you are back to the per-edit anti-pattern. Keep `PostToolUse` to a path append; keep all the expense at `Stop`.

## Related concepts

- [Session Accumulation Pattern](../concepts/session-accumulation-pattern.md) — the full three-hook choreography (`SessionStart` config + `PostToolUse` journal + `Stop` batch)
- [Hooks Adoption Ladder](../concepts/hooks-adoption-ladder.md) — source of the "never put tsc in PostToolUse" rule
- [Hook Automation Use Cases](../concepts/hook-automation-use-cases.md) — the recipe catalog this placement decision applies to
- [Synchronous vs async hook execution](./sync-vs-async-hooks.md) — the orthogonal axis: blocking vs background, not when to run
- [Hook Lifecycle Events](../concepts/hook-lifecycle-events.md) — the `PostToolUse` and `Stop` events this contrast attaches to
