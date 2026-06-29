---
type: index
created: 2026-06-29
updated: 2026-06-29
tags: [hooks, navigation, index, reference]
---

# Claude Code Hooks

Master navigation. Maintained by the indexer. Start with [overview](overview.md)
and the running [synthesis](synthesis.md).

## Sources

- [Hooks Guide](sources/clean/code-claude-com-docs-en-hooks-guide-md.md) — official Claude Code hooks tutorial (clean source)
- [Hooks Reference](sources/clean/code-claude-com-docs-en-hooks-md.md) — official hooks reference: event schemas, JSON I/O, decision control (clean source)

## Summaries

- [Hooks Guide](summaries/hooks-guide.md) — Automate actions with hooks (code.claude.com official guide)
- [Hooks Reference](summaries/hooks.md) — Event schemas, JSON I/O, and decision control (reference doc)

## Concepts

- [Hook Lifecycle Events](concepts/hook-lifecycle-events.md) — all 30+ events and when they fire
- [Hook Types](concepts/hook-types.md) — command, http, mcp_tool, prompt, agent
- [Hook Matchers](concepts/hook-matchers.md) — matcher field, `if` field, per-event filter table
- [Hook Exit Codes](concepts/hook-exit-codes.md) — exit 0/2/other and structured JSON output
- [Hook Scope](concepts/hook-scope.md) — configuration location and permission mode interactions
- [Hook Input and Output](concepts/hook-input-output.md) — JSON input envelope and output fields per event
- [Hook Decision Control](concepts/hook-decision-control.md) — per-event decision patterns reference

## Entities

## Comparisons

- [Hooks Guide accuracy check](comparisons/code-claude-com-docs-en-hooks-guide-md.md) — verified wiki pages against source; 2 inaccuracies corrected
- [matcher vs if field](comparisons/matcher-vs-if-field.md) — hook targeting levels: group-level name vs argument-level filtering
- [Sync vs async hooks](comparisons/sync-vs-async-hooks.md) — synchronous vs async: true vs asyncRewake: true execution modes

## Answers
