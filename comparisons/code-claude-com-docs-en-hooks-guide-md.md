---
type: comparison
title: "Comparison: hooks-guide source vs. wiki"
slug: code-claude-com-docs-en-hooks-guide-md
created: 2026-06-29
updated: 2026-06-29
tags: [hooks, accuracy, review]
sources:
  - sources/clean/code-claude-com-docs-en-hooks-guide-md.md
---

# Comparison: hooks-guide vs. wiki pages

Cross-check of `sources/clean/code-claude-com-docs-en-hooks-guide-md.md` against the wiki pages it produced. Verified and corrected after synthesis pass.

## Verified accurate

| Wiki claim | Source location | Status |
| :--------- | :-------------- | :----- |
| Core thesis quote (deterministic control layer) | Source intro | ✓ verbatim |
| Two-tier enforcement: deterministic vs. judgment hooks | Source intro | ✓ accurate |
| PreToolUse `deny` blocks even in `bypassPermissions` mode | "How hooks work" / permission mode section | ✓ accurate |
| Reverse does NOT hold: `allow` cannot override deny rules from settings | Same | ✓ accurate |
| `if` field fails open when Bash command can't be parsed | `if` field section | ✓ accurate |
| All matching hooks run in parallel; deduplication of identical commands | "How hooks work" | ✓ accurate |
| 7 recipes with correct event/matcher pairing | Recipe sections | ✓ accurate |
| Prompt hook timeout: 30s; agent hook timeout: 60s, 50 turns | Hook types section | ✓ accurate |
| `PermissionRequest` hooks do not fire in non-interactive (`-p`) mode | Limitations | ✓ accurate |
| `Stop` hooks have a block cap of 8 consecutive blocks | Limitations | ✓ accurate |
| `FileChanged` matcher uses literal pipe-separated filenames, not regex | "Reload environment" recipe | ✓ captured in matchers page |

## Inaccuracies corrected

### 1. Exit code description in `synthesis.md`

**Was**: "Exit code 1 — warning (shown in transcript, does not block)"

**Source says**: Any exit code **other than 0 or 2** causes the action to proceed, showing a hook error notice (first line of stderr) in the transcript. Exit code 1 is not special — it's just one of many non-0, non-2 codes that fall into this category.

**Correction applied**: `synthesis.md` updated to say "Other exit codes (non-0, non-2) — action proceeds, but the transcript shows a hook error notice with the first line of stderr."

Note: the `concepts/hook-exit-codes.md` page correctly described this as "other" — only the synthesis had the simplified "1" wording.

### 2. Event count in `synthesis.md` concept index

**Was**: "29 events with categories"

**Actual count**: 30 events in the `hook-lifecycle-events.md` table (SessionStart, Setup, UserPromptSubmit, UserPromptExpansion, PreToolUse, PermissionRequest, PermissionDenied, PostToolUse, PostToolUseFailure, PostToolBatch, Notification, MessageDisplay, SubagentStart, SubagentStop, TaskCreated, TaskCompleted, Stop, StopFailure, TeammateIdle, InstructionsLoaded, ConfigChange, CwdChanged, FileChanged, WorktreeCreate, WorktreeRemove, PreCompact, PostCompact, Elicitation, ElicitationResult, SessionEnd = 30).

**Correction applied**: `synthesis.md` concept index updated to "30 events with categories."

## Gaps (not yet in wiki)

- The `SubagentStart`/`SubagentStop` matcher supports agent type names (`general-purpose`, `Explore`, `Plan`, custom) — this is in the matchers page table but not discussed in depth.
- The Notification recipe table lists 6 matcher values (`permission_prompt`, `idle_prompt`, `auth_success`, `elicitation_dialog`, `elicitation_complete`, `elicitation_response`) — captured in the matchers table.
- `bypassPermissions` mode constraints (requires `--dangerously-skip-permissions` or equivalent at launch; cannot be persisted as `defaultMode`) — source note under auto-approve recipe. Not explicitly stated in the `hook-scope.md` permission mode section. Low priority; this is a permissions concern, not a hook-specific one.
- The `TeammateIdle` event has no matcher support — mentioned in the table but no description of when it fires. The lifecycle events page says "When an agent team teammate is about to go idle" — this is accurate.

## Open questions (for future sources to resolve)

See `synthesis.md` open questions section — all 5 remain unresolved with 1 source.
