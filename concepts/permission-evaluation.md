---
type: concept
title: "Permission Evaluation"
created: 2026-06-29
updated: 2026-06-29
tags: [permissions, rules, precedence, deny, hooks, matching]
source_count: 1
sources:
  - sources/clean/code-claude-com-docs-en-permissions-md.md
---

# Permission Evaluation

When Claude Code attempts a tool call, it resolves a decision — **allow, ask, or deny** — by matching the call against your [permission rules](./tool-permission-rules.md). This page describes that decision algorithm: the order rules are checked, how scopes combine, and how [hooks](./hook-decision-control.md) plug in.

## Evaluation order: deny, then ask, then allow

> Rules are evaluated in order: deny, then ask, then allow. The first match in that order determines the outcome, and rule specificity doesn't change the order.

Two consequences surprise people:

- **A deny rule cannot carry an allowlist exception.** A broad `Bash(aws *)` deny blocks every matching call, "including calls that also match a narrower allow rule like `Bash(aws s3 ls)`." The same holds between ask and allow: a matching `ask` rule prompts even when a more specific `allow` rule also matches.
- **Specificity is irrelevant.** Only the deny→ask→allow ordering matters; a longer pattern does not win over a shorter one in a different bucket.

## Bare tool name vs. scoped pattern

A deny rule behaves differently depending on whether it names a tool or scopes a pattern within one:

- **Bare name** (`Bash`, or the equivalent `Bash(*)`): removes the tool from Claude's context entirely, so "Claude never sees it."
- **Scoped pattern** (`Bash(rm *)`): leaves the tool available and blocks matching calls only when Claude attempts them.

Tool-name globs in the deny/ask position follow the bare-name behavior: `"*"` matches every tool and `"mcp__*"` matches every MCP tool, and a tool matched by a bare-name glob deny is removed from context.

## Matching by input parameter

Deny and ask rules can also match a top-level input parameter on any tool with `Tool(param:value)` — for example `Agent(model:opus)` or `Bash(run_in_background:true)`. Key constraints from the source:

- The parameter must be a **direct field** of the tool's input; nested fields are not matchable.
- Each rule names **one** parameter; gate on two by writing two rules.
- `*` is a wildcard; without it the match is exact. A parameter the model omits is never matched (`Agent(model:*)` does not match a call that leaves `model` unset).
- The value is compared against the **literal input before normalization** — `Agent(model:opus)` matches the alias but not a full model ID.
- Allow rules do **not** use this form; "An allow rule for one parameter value wouldn't establish that the call is safe overall," so allow rules keep each tool's own specifier syntax.

Fields a tool already canonicalizes — `command` (Bash/PowerShell), `file_path` (Read/Edit/Write), `path` (Grep/Glob), `notebook_path`, `url` (WebFetch) — are **not** matchable this way; `Bash(command:rm *)` is ignored with a startup warning. Use the tool's own specifier (`Bash(rm *)`) instead.

## Canonical names

The label shown in the transcript or permission dialog can differ from a tool's canonical name — e.g. `Stop Task` is canonically `TaskStop`. Rules and [hook matchers](./hook-matchers.md) match the **canonical name only**, so a rule written as `Stop Task` doesn't match. A deny/ask rule whose tool name matches no known tool produces a startup warning to catch typos (names containing `_` or `*` are exempt). Use the canonical names from the [tools reference](./built-in-tools.md).

## Precedence across scopes

Permission rules merge across [scopes](./configuration-scopes.md), but the deny-first rule wins globally:

1. **Managed settings** — can't be overridden by anything, including CLI arguments
2. **Command line arguments**
3. **Local project settings** (`.claude/settings.local.json`)
4. **Shared project settings** (`.claude/settings.json`)
5. **User settings** (`~/.claude/settings.json`)

> If a tool is denied at any level, no other level can allow it.

A user-level deny blocks a project-level allow, and vice versa, "because deny rules from any scope are evaluated before allow rules." Embedding hosts can add policy via the SDK `managedSettings` option with `parentSettingsBehavior: "merge"` — embedder values can tighten but not loosen policy.

## How hooks fit in

[PreToolUse hooks](./hook-decision-control.md) run **before** the permission prompt and can deny, force a prompt, or skip it. But hook decisions don't bypass rules:

- Deny and ask rules are evaluated **regardless** of what a hook returns — a matching deny blocks even when the hook returned `"allow"`, preserving deny-first precedence (including managed deny rules).
- A hook that **exits with code 2** stops the call before rules are evaluated, so it overrides even an allow rule. This is how you "run all Bash commands without prompts except for a few you want blocked": allow `"Bash"`, then reject specific commands in a PreToolUse hook.

## Related concepts

- [Tool Permission Rules](./tool-permission-rules.md) — the `ToolName(specifier)` format these rules use
- [Permission Settings](./permission-settings.md) — the `allow`/`ask`/`deny` blocks
- [Permission Modes](./permission-modes.md) — the session preset evaluated around these rules
- [Hook Decision Control](./hook-decision-control.md) — PreToolUse hooks and exit-code-2 blocking
- [Settings Precedence](./settings-precedence.md) — the general scope ordering
- [Bash Permission Matching](./bash-permission-matching.md) — how Bash specifiers are matched
- [File Permission Patterns](./file-permission-patterns.md) — how Read/Edit path specifiers are matched
