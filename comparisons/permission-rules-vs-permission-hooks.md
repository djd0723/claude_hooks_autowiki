---
type: comparison
title: "Permission rules vs. permission hooks"
created: 2026-06-29
updated: 2026-06-29
tags: [permissions, hooks, deny, pretooluse, permissionrequest, decision, security]
sources:
  - sources/clean/code-claude-com-docs-en-permissions-md.md
  - sources/clean/claudefa-st-blog-tools-hooks-permission-hook-guide.md
---

# Permission rules vs. permission hooks

Two different machines decide whether a tool call runs: the **static permission rules**
(`allow`/`ask`/`deny` in settings) and **permission hooks** (a `PreToolUse` or
`PermissionRequest` handler that returns a decision programmatically). They look like
substitutes — both can stop a `Bash(rm *)` — but they sit at different points in the same
pipeline, and conflating them is how people build controls that don't hold. This page maps when
to reach for each and the one asymmetry that governs every interaction between them.

## The pipeline order

A permission hook runs **before** the static rules are consulted, but it does not replace them:

> "PreToolUse hooks run **before** the permission prompt and can deny, force a prompt, or skip it. But hook decisions don't bypass rules."

So the evaluation is layered, not either/or: the hook gets first say, then the static rules
([permission evaluation](../concepts/permission-evaluation.md)) get the final say in the
restrictive direction.

## The asymmetry that decides everything: a hook narrows, never widens

This is the load-bearing rule. A hook can **add** restriction on top of the static rules, but
it can never remove a restriction the rules impose:

> "Deny and ask rules are evaluated **regardless** of what a hook returns — a matching deny
> blocks even when the hook returned `\"allow\"`, preserving deny-first precedence (including
> managed deny rules)."

Combined with the global deny invariant —

> "If a tool is denied at any level, no other level can allow it."

— the consequence is unambiguous: **a permission hook is a one-way valve toward more
restriction.** A hook that says "allow" is only honoured for calls the rules would have asked
about or allowed anyway; it cannot rescue a call a `deny` rule (at any scope, especially a
managed one) already kills. This is exactly why the [AI permission reviewer](../concepts/ai-permission-reviewer.md)
case study is built as a *hook* and not a settings generator — it can only tighten the floor the
permission system already sets.

The one place a hook gets the last word is the **hard block**:

> "A hook that **exits with code 2** stops the call before rules are evaluated, so it overrides
> even an allow rule."

So the full ordering is: **hook exit-2 block → static deny → static ask → (hook decision, if
allow/ask) → static allow.** A hook can veto an allow; it cannot veto a deny.

## When to reach for static rules

Static `allow`/`ask`/`deny` rules are the default and should carry most of the load. Use them
when:

- **The decision is a fixed predicate** on the tool name or its specifier — `Bash(git *)`,
  `Read(./secrets/**)`, `Agent(model:opus)`. No process to launch, no latency, evaluated in the
  hot path for free.
- **You need a guarantee that cannot be softened** — a `deny` rule is absolute and global, and a
  managed-scope deny "can't be overridden by anything, including CLI arguments." A hook cannot
  provide this; a hook's "deny" is conditional on the hook actually running and returning.
- **You want a tool removed from Claude's context entirely** — a bare-name deny (`Bash`, `"*"`,
  `"mcp__*"`) does this so "Claude never sees it." A hook cannot remove a tool from context; it
  only intercepts calls Claude already decided to attempt.

The canonical recipe — *allow broadly, deny specifically* — lives entirely in static rules:
allow `"Bash"`, then block the handful you never want. The source frames a hook as the way to do
the "block the handful" half with logic instead of a static list:

> "This is how you 'run all Bash commands without prompts except for a few you want blocked':
> allow `\"Bash\"`, then reject specific commands in a PreToolUse hook."

## When to reach for a permission hook

A hook earns its latency and failure surface only when a static rule **can't express the
decision**:

- **The decision needs context a pattern can't see** — recent conversation, working directory,
  the *intent* behind a command. The ClaudeFast product's framing is the motivating case: it
  routes "ambiguous operations" to "a fast, cheap LLM" that receives the tool, command, working
  directory, and recent context, because `docker system prune -af` is fine in one context and a
  mistake in another.
- **You want to replace the interactive prompt with an automated approver** — a
  `PermissionRequest` hook can "approve or deny programmatically instead of showing a dialog,"
  which is the whole point of the [permission-hook pattern](../concepts/ai-permission-reviewer.md):
  Claude runs uninterrupted, dangerous calls are blocked, edge cases get judged.
- **You need to log, transform, or audit** the call as a side effect of the decision —
  behaviour a static rule has no place to put.

The standing caution from the case study is that a judgment hook is only worth it for the
residual gray area. Its three-tier funnel pushes most traffic through **deterministic** allow/
deny tiers precisely because the LLM tier has cost and latency — "most operations hit Tier 1 or
2." The deterministic tiers there are doing the same job a static rule does; the hook exists for
the part a rule can't.

## Decision table

| Situation | Reach for |
| :-------- | :-------- |
| Fixed rule on tool name / specifier / input param | **Static rule** ([tool permission rules](../concepts/tool-permission-rules.md)) |
| Must be unbreakable / managed-enforced | **Static `deny`** (a hook can't guarantee it) |
| Remove a tool from Claude's context | **Bare-name `deny`** |
| Decision depends on conversation/cwd/intent | **Permission hook** |
| Auto-approve to avoid prompt fatigue | **`PermissionRequest` hook** |
| Catastrophe kill-switch that beats even an allow | **`PreToolUse` hook, exit code 2** |
| Block-most, allow-a-few, statically | **Static rules** (`allow` broad, `deny` specific) |
| Block-most, allow-a-few, with logic | **Static `allow` + a `PreToolUse` deny hook** |

## The trap to avoid

Do not use a hook's `"allow"` to try to *loosen* policy — to "approve" something a `deny` rule
or a stricter scope forbids. It silently won't work: the deny is re-evaluated after the hook and
wins. If you find yourself wanting a hook to widen access, the policy belongs in the static rules
(or a less restrictive scope), not in a hook. Hooks are for *narrowing* and *judging*, never for
*overriding* a restriction.

## Related

- [Permission Evaluation](../concepts/permission-evaluation.md) — the deny→ask→allow algorithm and the "how hooks fit in" rules quoted here
- [AI Permission Reviewer](../concepts/ai-permission-reviewer.md) — a worked permission-hook case study (three-tier funnel)
- [Permission Modes](../concepts/permission-modes.md) — the native, model-free alternative to `--dangerously-skip-permissions`
- [Hook Decision Control](../concepts/hook-decision-control.md) — the allow/deny/exit-2 mechanics a permission hook returns
- [Sync vs. Async Hooks](./sync-vs-async-hooks.md) — why a permission decision must be synchronous (async hooks can't block)
- [Hooks Adoption Ladder](../concepts/hooks-adoption-ladder.md) — where judgment hooks sit on the risk ladder
