---
type: synthesis
created: 2026-06-29
updated: 2026-06-29
tags: [hooks, automation, lifecycle, determinism, extension, json-schema, decision-control]
source_count: 2
sources:
  - sources/clean/code-claude-com-docs-en-hooks-guide-md.md
  - sources/clean/code-claude-com-docs-en-hooks-md.md
---

# Claude Code Hooks — Synthesis

The running thesis: what the sources, taken together, add up to. Revised by the
synthesizer after each source — through-lines, where sources agree, and where they
contradict each other.

---

## Core thesis (2 sources)

Claude Code hooks exist to solve a fundamental reliability problem in LLM-driven tooling: **you cannot guarantee that a language model will always do a thing**. Hooks provide the deterministic layer that sits alongside the model — running shell commands, spawning subagents, or calling HTTP endpoints at fixed lifecycle points regardless of what the model decides. The official source states it directly:

> "They provide deterministic control over Claude Code's behavior, ensuring certain actions always happen rather than relying on the LLM to choose to run them."

The design philosophy is additive, not adversarial: hooks do not replace the LLM; they wrap it at seams (before, after, on notification) to enforce policies the LLM might otherwise skip.

---

## Through-lines confirmed across both sources

### 1. Hooks as a two-tier enforcement model

There are two distinct modes of hook enforcement:

- **Deterministic command hooks** — shell commands or HTTP calls that apply fixed rules (format this file, block this path, log this event). No LLM involved; exit code and stdout alone determine the outcome.
- **Judgment-bearing hooks** — `type: prompt` and `type: agent` delegate a decision to a Claude model (Haiku by default) or a subagent. These exist for cases where a rule cannot be expressed as a simple pattern.

This split matters: using a judgment hook where a deterministic one would do adds latency, API cost, and a failure mode (model refusal, timeout). Using a deterministic hook where judgment is needed is brittle. Both sources treat the distinction as a design decision the user makes per hook.

### 2. Exit codes are the primary communication channel

Hooks talk back to Claude Code through:
- **Exit code 0** — success, no action needed
- **Other exit codes** (non-0, non-2) — action proceeds, but the transcript shows a hook error notice with the first line of stderr
- **Exit code 2** — hard block (tool call denied, model is told to continue without it)
- **Stdout JSON** — structured `hookSpecificOutput` to pass decisions, reasons, or directives to the model

The reference (source 2) extends this: `asyncRewake: true` on a command hook runs in the background and wakes Claude on exit code 2, showing stderr as a system reminder so Claude can react to long-running background failures. This is a distinct pattern from synchronous blocking.

### 3. Permission system interaction: hooks fire before permission checks

A `PreToolUse` hook that returns `permissionDecision: "deny"` blocks even in `bypassPermissions` mode. The reverse does not hold: a hook `"allow"` cannot override a `deny` rule in settings. This establishes hooks as **at least as powerful as the permission system for blocking**, but subordinate to it for allowing.

The reference confirms and extends this with enterprise scope: `allowManagedHooksOnly` blocks user/project/plugin hooks entirely, and `disableAllHooks` in user/project settings cannot override managed-policy hooks. Enterprise policy sits above all other scope levels.

### 4. The `if` field extends matchers from name-level to argument-level

The `matcher` field filters by tool name (coarse). The `if` field (v2.1.85+) filters by tool name **and** arguments — `"if": "Bash(git *)"` fires only on git commands, not every Bash call. The `if` filter **fails open**: if the tool arguments can't be parsed, the hook runs anyway. This is a safety choice (prefer running the hook over silently skipping it) but callers should not rely on it as a soft filter for hard security decisions — use the permission system for those.

The reference (source 2) adds that matcher rules vary per event: for `SessionStart`, matchers target startup reason (`startup`, `resume`, `clear`, `compact`); for `Notification`, they target notification type; for `FileChanged`, they match literal filenames. Several events (e.g. `UserPromptSubmit`, `PostToolBatch`, `Stop`) accept no matcher at all.

### 5. Configuration is hierarchical and additive, not overriding

Hooks can live in `~/.claude/settings.json` (user), `.claude/settings.json` (project), or inside plugins and skills. All matching hooks for an event run **in parallel** — there is no "last wins" or "first wins" override. The most restrictive `PreToolUse` decision wins across hooks. This means multiple hooks stacking from different scopes is normal and expected, not a footgun.

### 6. Event decision patterns are diverse and event-specific (source 2)

Different events use structurally different decision mechanisms. There is no single `decision` field — the shape of the blocking output depends on which event fired:

- Most events use top-level `{"decision": "block", "reason": "..."}` in JSON output
- `PreToolUse` uses `hookSpecificOutput.permissionDecision` with `allow`/`deny`/`ask`/`defer`
- `PermissionRequest` uses `hookSpecificOutput.decision.behavior`
- `PermissionDenied` uses `hookSpecificOutput.retry: true` to grant a retry
- `MessageDisplay` uses `hookSpecificOutput.displayContent` to replace rendered text (does not affect transcript)
- The `defer` mechanism (`permissionDecision: "defer"`) is exclusive to Agent SDK / `claude -p` integrations and pauses the session for external input

When multiple `PreToolUse` hooks conflict, precedence is: `deny > defer > ask > allow`.

### 7. Universal JSON input fields travel with every hook (source 2)

Every hook event receives a common envelope on stdin: `session_id`, `transcript_path`, `cwd`, `permission_mode`, `effort` (with `level`), `hook_event_name`, and `agent_id`/`agent_type` when inside a subagent. This means any hook can inspect the session context without any configuration — provenance, permission mode, and effort are always available.

### 8. The `terminalSequence` output field enables terminal integration (source 2)

Hooks cannot write to `/dev/tty` directly, but can emit OSC escape sequences (OSC 0/1/2/9/99/777, BEL) via the `terminalSequence` field in JSON output. Claude Code emits these through its own terminal path. This enables hooks to set terminal titles, trigger notifications, or communicate with terminal multiplexers without requiring raw tty access.

---

## Resolved open questions

**Q1 (async hooks):** The reference fully documents async hooks. `async: true` runs in the background without blocking. `asyncRewake: true` also wakes Claude on exit code 2. Async hooks are a first-class feature, not an edge case.

**Q3 (matcher/`if` distinction):** The reference clarifies that matcher targets vary by event type — the same `matcher` field hits different dimensions depending on which event fired. The `if` field (argument-level filter) applies only to tool-use events. This is a non-trivial distinction that tip sources may simplify or omit.

---

## Open questions for future sources to resolve

1. **How do practitioner sources characterize the `type: agent` hook?** The official reference says 60-second timeout, 50 tool-use turns. Do real-world guides treat this as sufficient for meaningful verification, or too tight?

2. **What real-world hook recipes appear in the wild?** The official guide offers 7 examples. Community sources likely surface additional patterns (pre-commit enforcement, PR labeling, CI triggering, self-improving CLAUDE.md).

3. **Scope conflicts**: do plugin/skill hooks collide with user hooks in documented ways? The reference covers enterprise scope well, but practitioner collisions at the user/project/plugin boundary are uncharted.

4. **`MessageDisplay` in practice**: the event is display-only and fires while text streams. Do practitioners use it for redaction or transformation in ways the reference doesn't anticipate?

5. **`defer` in Agent SDK integrations**: the mechanism is documented but complex. Do tip sources offer implementation patterns for the pause/resume flow?

---

## Concept index (pages so far)

- [Hook Lifecycle Events](concepts/hook-lifecycle-events.md) — 30+ events with cadences and categories
- [Hook Types](concepts/hook-types.md) — command, http, mcp_tool, prompt, agent handler fields
- [Hook Matchers](concepts/hook-matchers.md) — matcher field, `if` field, per-event filter table
- [Hook Exit Codes](concepts/hook-exit-codes.md) — exit codes and per-event blocking table
- [Hook Scope](concepts/hook-scope.md) — config locations, permission mode interactions, enterprise scope
- [Hook Input/Output](concepts/hook-input-output.md) — JSON input envelope, exit codes, JSON output fields
- [Hook Decision Control](concepts/hook-decision-control.md) — per-event decision patterns reference
- [Summary: Hooks Guide](summaries/hooks-guide.md) — setup recipes, troubleshooting, examples
- [Summary: Hooks Reference](summaries/hooks.md) — schemas, JSON I/O, decision control patterns
