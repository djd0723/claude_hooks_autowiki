---
type: concept
title: "Tool Permission Rules"
created: 2026-06-29
updated: 2026-06-29
tags: [tools, permissions, rules, syntax, subagents, hooks, skills]
source_count: 1
sources:
  - sources/clean/code-claude-com-docs-en-tools-reference-md.md
---

# Tool Permission Rules

You reference [built-in tool](./built-in-tools.md) names directly when defining permissions and other configuration. The same rule format — **`ToolName(specifier)`** — is used everywhere, but the *specifier* depends on the tool, and several tools share one format.

## Where the same rule format is used

The `ToolName(specifier)` syntax is accepted in six places:

- `permissions.allow` and `permissions.deny` in [settings](./permission-settings.md), and the `/permissions` interface
- the `--allowedTools` and `--disallowedTools` CLI flags
- the Agent SDK's `allowedTools` and `disallowedTools` options
- a [subagent's `tools` or `disallowedTools`](./subagent-tool-access.md) frontmatter
- a [skill's `allowed-tools`](./skill-frontmatter.md) frontmatter
- a hook's [`if` condition](./hook-matchers.md)

## Specifier format by tool

> The specifier depends on the tool, and several tools share a format.

| Rule format | Applies to | Specifier kind |
| :---------- | :--------- | :------------- |
| `Bash(npm run *)` | Bash, **Monitor** | Command pattern matching |
| `PowerShell(Get-ChildItem *)` | PowerShell | Command pattern matching |
| `Read(~/secrets/**)` | Read, **Grep, Glob, LSP** | Path pattern matching |
| `Edit(/src/**)` | Edit, **Write, NotebookEdit** | Path pattern matching |
| `Skill(deploy *)` | Skill | Skill name matching |
| `Agent(Explore)` | Agent | Subagent type matching |
| `WebFetch(domain:example.com)` | WebFetch | Domain matching |
| `WebSearch` | WebSearch | No specifier — allow/deny the whole tool |

Note that one rule can cover several tools: a `Read(...)` rule governs `Read`, `Grep`, `Glob`, and `LSP`; an `Edit(...)` rule governs `Edit`, `Write`, and `NotebookEdit`; a `Bash(...)` rule also governs `Monitor`.

Tools not listed above — such as `ExitPlanMode` or `ShareOnboardingGuide` — accept **only the bare tool name with no specifier**.

## Two rules that surprise people

- **`Edit` implies `Read`.** "An `Edit(...)` allow rule also grants read access to the same path, so you don't need a matching `Read(...)` rule."
- **Hook matchers are not rules.** Hook `matcher` fields use **bare tool names**, not the parenthesized `ToolName(specifier)` format. The parenthesized format is only for the `if` condition and the permission/CLI/SDK/frontmatter surfaces above. For the field names each tool passes to `tool_input` in hooks, see the [hook input reference](./hook-input-output.md).

## Related concepts

- [Built-in Tools](./built-in-tools.md) — the catalog of tool names these rules reference
- [Permission Evaluation](./permission-evaluation.md) — how these rules resolve to allow/ask/deny, plus parameter and tool-name matching
- [Bash Permission Matching](./bash-permission-matching.md) — how `Bash(...)` specifiers match commands, wrappers, and compounds
- [File Permission Patterns](./file-permission-patterns.md) — how `Read(...)`/`Edit(...)` path specifiers match
- [Permission Settings](./permission-settings.md) — the `allow`/`ask`/`deny` blocks and merge behavior
- [Subagent Tool Access](./subagent-tool-access.md) — `tools` / `disallowedTools` precedence on subagents
- [Skill Frontmatter](./skill-frontmatter.md) — the `allowed-tools` field
- [Hook Matchers](./hook-matchers.md) — bare-name matchers and the argument-level `if` field
