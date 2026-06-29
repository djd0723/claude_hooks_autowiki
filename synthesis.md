---
type: synthesis
created: 2026-06-29
updated: 2026-06-29
tags: [hooks, automation, lifecycle, determinism, extension]
source_count: 1
sources:
  - sources/clean/code-claude-com-docs-en-hooks-guide-md.md
---

# Claude Code Hooks — Synthesis

The running thesis: what the sources, taken together, add up to. Revised by the
synthesizer after each source — through-lines, where sources agree, and where they
contradict each other.

---

## Core thesis (1 source)

Claude Code hooks exist to solve a fundamental reliability problem in LLM-driven tooling: **you cannot guarantee that a language model will always do a thing**. Hooks provide the deterministic layer that sits alongside the model — running shell commands, spawning subagents, or calling HTTP endpoints at fixed lifecycle points regardless of what the model decides. The official source states it directly:

> "They provide deterministic control over Claude Code's behavior, ensuring certain actions always happen rather than relying on the LLM to choose to run them."

The design philosophy is additive, not adversarial: hooks do not replace the LLM; they wrap it at seams (before, after, on notification) to enforce policies the LLM might otherwise skip.

---

## Through-lines established by source 1

### 1. Hooks as a two-tier enforcement model

There are two distinct modes of hook enforcement:

- **Deterministic command hooks** — shell commands or HTTP calls that apply fixed rules (format this file, block this path, log this event). No LLM involved; exit code and stdout alone determine the outcome.
- **Judgment-bearing hooks** — `type: prompt` and `type: agent` delegate a decision to a Claude model (Haiku by default) or a subagent. These exist for cases where a rule cannot be expressed as a simple pattern.

This split matters: using a judgment hook where a deterministic one would do adds latency, API cost, and a failure mode (model refusal, timeout). Using a deterministic hook where judgment is needed is brittle. The guide treats the distinction as a design decision the user makes per hook.

### 2. Exit codes are the primary communication channel

Hooks talk back to Claude Code through:
- **Exit code 0** — success, no action needed
- **Exit code 1** — warning (shown in transcript, does not block)
- **Exit code 2** — hard block (tool call denied, model is told to continue without it)
- **Stdout JSON** — structured `hookSpecificOutput` to pass decisions, reasons, or directives to the model

This is a clean, shell-idiomatic protocol. It means hooks are composable with any tool that produces an exit code — linters, test runners, git commands — without wrapper logic.

### 3. Permission system interaction: hooks fire before permission checks

A `PreToolUse` hook that returns `permissionDecision: "deny"` blocks even in `bypassPermissions` mode. The reverse does not hold: a hook `"allow"` cannot override a `deny` rule in settings. This establishes hooks as **at least as powerful as the permission system for blocking**, but subordinate to it for allowing. Future tip sources should be checked for whether they misstate this.

### 4. The `if` field extends matchers from name-level to argument-level

The `matcher` field filters by tool name (coarse). The `if` field (v2.1.85+) filters by tool name **and** arguments — `"if": "Bash(git *)"` fires only on git commands, not every Bash call. The `if` filter **fails open**: if the tool arguments can't be parsed, the hook runs anyway. This is a safety choice (prefer running the hook over silently skipping it) but callers should not rely on it as a soft filter for hard security decisions — use the permission system for those.

### 5. Configuration is hierarchical and additive, not overriding

Hooks can live in `~/.claude/settings.json` (user), `.claude/settings.json` (project), or inside plugins and skills. All matching hooks for an event run **in parallel** — there is no "last wins" or "first wins" override. The most restrictive `PreToolUse` decision wins across hooks. This means multiple hooks stacking from different scopes is normal and expected, not a footgun.

---

## Open questions for future sources to resolve

1. **How do tip sources characterize the `type: agent` hook?** The official source says 60-second timeout, 50 tool-use turns. Do practitioner guides treat this as a realistic budget or too tight for real verification?

2. **What real-world hook recipes appear in the wild?** The official guide offers 7. Community sources likely surface additional patterns (pre-commit enforcement, PR labeling, CI triggering, self-improving CLAUDE.md).

3. **Is the `matcher`/`if` distinction well understood?** There is potential for confusion between tool-name matchers and argument-level filters. Do tip sources clarify or muddy this?

4. **Scope conflicts**: do plugin/skill hooks ever collide with user hooks in ways the official guide doesn't cover?

5. **Async hooks**: the official guide mentions them in the reference but does not explain them here. Do tip sources document async hook usage?

---

## Concept index (pages so far)

- [Hook Lifecycle Events](concepts/hook-lifecycle-events.md) — 29 events with categories
- [Hook Types](concepts/hook-types.md) — command, http, mcp_tool, prompt, agent
- [Hook Matchers](concepts/hook-matchers.md) — matcher field, `if` field, per-event filter table
- [Hook Exit Codes](concepts/hook-exit-codes.md) — exit codes and structured JSON output
- [Hook Scope](concepts/hook-scope.md) — config locations and permission mode interactions
- [Summary: Hooks Guide](summaries/hooks-guide.md) — full source summary with recipes
