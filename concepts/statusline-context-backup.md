---
type: concept
title: "StatusLine-Driven Context Backup (Context Recovery)"
created: 2026-06-29
updated: 2026-06-29
tags: [hooks, statusline, context-window, compaction, backup, recovery]
source_count: 1
sources:
  - sources/clean/claudefa-st-blog-tools-hooks-hooks-guide.md
---

# StatusLine-Driven Context Backup (Context Recovery)

The reference pages treat [`PreCompact`](./hook-lifecycle-events.md) as *the* place to save state before the context window collapses. This page captures a practitioner's argument for a different — and earlier — trigger: the **StatusLine**. It's a self-contained recovery pattern (the "Context Recovery Hook") that the official hook references don't surface, mined from one engineer's production setup.

## Why StatusLine, not PreCompact

`PreCompact` only fires *at* compaction — by then you're already losing context. StatusLine runs continuously and is the one place live context metrics are exposed:

> "StatusLine is the only mechanism that receives live context metrics."

So it can back up **proactively**, before the cliff:

> "Unlike PreCompact (which triggers on compaction), StatusLine-based backups capture state proactively."

It's wired in like any other status-line command:

```json
{
  "statusLine": {
    "type": "command",
    "command": "node \"$CLAUDE_PROJECT_DIR/.claude/hooks/ContextRecoveryHook/statusline-monitor.mjs\""
  }
}
```

The `$CLAUDE_PROJECT_DIR` prefix is load-bearing, not cosmetic:

> "The `$CLAUDE_PROJECT_DIR` prefix points the hook at your project root from any working directory; a bare relative path throws MODULE_NOT_FOUND once a session's working directory moves into a subdirectory."

(See [working directories](./working-directories.md) for why a session's cwd drifts.)

## The 33K autocompact-buffer trap

The metric StatusLine hands you is *not* the real headroom. The `remaining_percentage` field already reserves a fixed buffer for autocompaction, so naive thresholding fires far too late. Subtract it first:

> "The `remaining_percentage` field includes a fixed 33K-token autocompact buffer. To get actual 'free until autocompact'":

```js
const AUTOCOMPACT_BUFFER_TOKENS = 33000;
const autocompactBufferPct = (AUTOCOMPACT_BUFFER_TOKENS / windowSize) * 100;
const freeUntilCompact = Math.max(0, pctRemainTotal - autocompactBufferPct);
```

Everything downstream thresholds against `freeUntilCompact`, not the raw percentage.

## Dual trigger system: tokens primary, percent as safety net

Two trigger systems run simultaneously. The **token-based** system is primary; **percentage** thresholds are a backstop.

```js
// Token-based triggers (primary - works across all window sizes)
const TOKEN_FIRST_BACKUP = 50000;     // First backup at 50k tokens used
const TOKEN_UPDATE_INTERVAL = 10000;  // Update every 10k tokens after

// Percentage-based triggers (safety net, especially for 200k windows)
const THRESHOLDS = [30, 15, 5];
```

The reason tokens lead and percent follows is window size:

> "The token-based system ensures early backups on large context windows (1M) where percentage thresholds would fire too late."

On a 1M-token window, "15% free" is still 150K tokens of runway — a percentage trigger would sleep through the entire useful window. Absolute token counts fire the same on a 200K and a 1M window.

## Architecture: three files, one source of truth

The system separates monitoring, triggering, and the backup logic itself:

```
.claude/hooks/ContextRecoveryHook/
├── backup-core.mjs        # Shared backup logic (parsing, formatting, saving)
├── statusline-monitor.mjs # Threshold detection + display (calls backup-core)
└── conv-backup.mjs        # PreCompact trigger (calls backup-core)
```

| File | Trigger | Responsibility |
| :--- | :--- | :--- |
| `backup-core.mjs` | Called by others | Parse transcript, format markdown, save file, update state |
| `statusline-monitor.mjs` | StatusLine (continuous) | Monitor tokens/context %, detect triggers, display status |
| `conv-backup.mjs` | `PreCompact` hook | Handle pre-compaction event |

The payoff is single-point maintenance:

> "This architecture means backup logic lives in one place. Changes to formatting, file naming, or state management only need to be made in backup-core.mjs."

Note both the continuous StatusLine path *and* the `PreCompact` path call the same core — StatusLine is the proactive primary, `PreCompact` the last-ditch catch.

## Files, display, and shared state

Backups are numbered with timestamps so a session keeps a history:

```
.claude/backups/1-backup-26th-Jan-2026-4-30pm.md
.claude/backups/2-backup-26th-Jan-2026-5-15pm.md
```

The status line then tells you exactly which file to reload:

```
[!] 25.0% free (50.0K/200K)
-> .claude/backups/3-backup-26th-Jan-2026-5-45pm.md
```

Both the StatusLine and `PreCompact` paths reconcile through one shared state file at `~/.claude/claudefast-statusline-state.json`:

```json
{
  "sessionId": "abc123",
  "lastFreeUntilCompact": 25.5,
  "currentBackupPath": ".claude/backups/3-backup-26th-Jan-2026-5-45pm.md"
}
```

## The recovery workflow

The backup is only half the value — the source prescribes how to *use* it after the window collapses:

> "When compaction occurs, run `/clear` to start a fresh session, then load the backup file shown in the statusline. This avoids confusion from having both compaction summary and injected context."

The deliberate move is to discard the auto-generated compaction summary entirely (via `/clear`) and reload the clean, self-authored backup instead, rather than working from two overlapping context sources.

## Related concepts

- [Hook Lifecycle Events](./hook-lifecycle-events.md) — `PreCompact`, the reactive trigger this pattern front-runs
- [Working Directories](./working-directories.md) — why a bare relative path breaks once the session's cwd moves, mandating `$CLAUDE_PROJECT_DIR`
- [Hooks Adoption Ladder (Practitioner Playbook)](./hooks-adoption-ladder.md) — the same source's broader case for moving guarantees out of `CLAUDE.md` and into deterministic hooks
- [Hook Automation Use Cases](./hook-automation-use-cases.md) — where transcript-backup sits among the generic recipes
