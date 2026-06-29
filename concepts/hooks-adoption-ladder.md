---
type: concept
title: "Hooks Adoption Ladder (Practitioner Playbook)"
created: 2026-06-29
updated: 2026-06-29
tags: [hooks, patterns, case-study, workflow, security, verification, latency]
source_count: 1
sources:
  - sources/clean/thomas-wiegold-com-blog-claude-code-hooks.md
---

# Hooks Adoption Ladder (Practitioner Playbook)

The reference pages ([hook lifecycle events](./hook-lifecycle-events.md), [hook types](./hook-types.md), [hook decision control](./hook-decision-control.md)) describe the hook *surface*. This page captures a practitioner's field-tested argument for **why** and **in what order** to adopt hooks — the gold-nuggets from one engineer who spent weeks migrating his workflow out of `CLAUDE.md` and into hooks.

## The central thesis: guarantees, not suggestions

The framing that organizes everything else: a `CLAUDE.md` instruction is advisory and the model may ignore it; a hook is deterministic and always fires.

> "Claude Code hooks let you wire deterministic shell commands and LLM evaluators into specific points in the agent's lifecycle. Not suggestions. Guarantees. The thing that's supposed to happen, happens, every time, without the model deciding whether to bother."

The practical decision heuristic that falls out of this:

> "if you're typing 'Claude must always...' or 'Claude should never...' in your CLAUDE.md, that's a hook. If you're typing 'When working on X, prefer Y', that's a skill."

And the blunt version of the value proposition:

> "Three hooks that always run beat 30 pages of advisory documentation Claude might or might not follow."

This is the deterministic-layer mindset that the [hook automation use cases](./hook-automation-use-cases.md) recipes assume but never state outright. (Compare the same source's "must/never → hook, when/prefer → skill" split against [skill invocation control](./skill-invocation-control.md).)

## The four-stage ladder

The source's spine is a staged adoption order — each stage is independently shippable, and the leverage increases as you climb:

1. **Stage 1 — Format & lint on every edit.** A `PostToolUse` hook matching `Edit|Write` that formats and autofixes the touched file. "Every project gets this hook on day one." The unsexy upside: PR review signal-to-noise jumps because you stop reading auto-formatted diffs and "Claude stops 'fixing' formatting that wasn't broken."
2. **Stage 2 — Security guardrails.** A user-global `PreToolUse` `Bash` hook that denies dangerous command patterns (see *Safe YOLO* below).
3. **Stage 3 — Logging & observability.** A no-matcher `PostToolUse` hook appending each tool call to a JSONL audit trail (`async: true` so it stays off the hot path). The rationale: "there's no other tamper-evident way to capture what Claude actually did across long sessions" — the transcript is local, single-session, and gets compacted.
4. **Stage 4 — Forced verification with `Stop` hooks.** The stage the author calls the one that "genuinely changed how I work with Claude" (see *The verification escalation ladder* below).

Stages 1–3 are guards and observers; Stage 4 is the one that lets you "stop reading 'I've completed the task' and start trusting it. Or rather, you don't trust it. The hook does."

## Safe YOLO: deny survives bypass mode

The single most load-bearing insight for security-conscious autonomy:

> "a PreToolUse hook returning permissionDecision: 'deny' blocks the tool even under --dangerously-skip-permissions."

Because:

> "bypass mode disables the interactive prompts and the auto-mode classifier. It does not disable hooks."

So you can run in a sandboxed worktree with `--dangerously-skip-permissions` for speed while a `PreToolUse` guard still has the final word — the community's "Safe YOLO". This *generalizes* the built-in circuit breakers documented under [permission modes](./permission-modes.md) (which always prompt on `rm -rf /` and `rm -rf ~`): a deny-hook lets you author your *own* circuit breakers beyond the two built in.

Why guards beat the model's own judgment here:

> "The model genuinely doesn't understand the OS-level consequences of path expansion."

When Claude emits `rm -rf $TARGET_DIR` with `TARGET_DIR=~/proj`, the model can't see that the variable was unset upstream — but the hook sees the *expanded* form and stops it. (The source cites real incidents: GitHub #10077, where Claude ran `rm -rf` on a home directory, and #12637, a literal `~` directory whose glob expansion wiped the real home — both in *standard* permission mode, not bypass.) See [hook decision control](./hook-decision-control.md) for the `permissionDecision: deny` mechanics.

## Latency lives in the hot path

A performance discipline absent from the reference docs:

> "hook latency lives in Claude's hot path. A hook that takes 3 seconds adds 3 seconds to every single tool call."

Concrete consequences the source draws:

- **Never put `tsc --noEmit` in `PostToolUse`** — "the most common trap I see." Typecheck is 10–30s; multiply by ~50 edits per feature and you've added ~25 minutes of wall-clock for errors the Stage-4 `Stop` hook catches anyway.
- **Prefer fast tooling** (Rust-based `oxfmt`/`oxlint`) precisely *because* the hook is on every tool call.
- **Make logging `async: true`** so the audit hook never blocks the agent loop — "you'll have hundreds of these per session."
- **Trim verifier output** (`tail -50`) so failure feedback stays cheap on the context window.

This is the cost lens on [hook exit codes and output](./hook-exit-codes.md) and the `async` flag in [hook types](./hook-types.md). For how to escape the `tsc`-in-`PostToolUse` trap without losing per-edit reactivity, see [per-edit hook processing vs. session-batch processing](../comparisons/per-edit-processing-vs-session-batch.md).

## The verification escalation ladder

Stage 4's deeper lesson: forced verification has three rungs of increasing cost and judgment, and you climb only as far as the task demands.

1. **Deterministic shell-out** (`command` hook) — runs your real `tsc`/test commands; blocks with `{"decision":"block","reason":...}` on failure. Catches "the mechanical stuff."
2. **`prompt` hook (Haiku evaluator)** — re-reads the conversation and changed files and judges whether the implementation *actually matches the request*, returning `{"ok": true}` / `{"ok": false, "reason": "..."}`. "About a tenth of a cent per fire." Catches "Claude implemented the wrong scoring algorithm because it misunderstood the spec" — which tests can't.
3. **`agent` hook (subagent verifier)** — experimental, ~60s default timeout, up to 50 tool turns; reserved for heavy cases like overnight-refactor verification "because the cost is real."

The non-negotiable gotcha that makes Stage 4 safe:

> "Always check stop_hook_active at the top of the script and exit 0 when it's true."

Without it the first failed gate becomes an infinite loop. The `prompt`/`agent` rungs are the practitioner counterparts to the LLM-evaluator handler types in [hook types](./hook-types.md).

## Ship-blocking gotchas

Field-learned traps, most of which the reference pages don't surface:

- **`PostToolUse` can't undo.** "By the time it fires, the file is already written. Use PreToolUse for prevention, PostToolUse for reaction."
- **Last-write-wins on `updatedInput`.** If two `PreToolUse` hooks rewrite the same tool field, order is non-deterministic — "don't have two hooks fighting over the same field."
- **`additionalContext` caps at 10,000 chars and stales on resume.** Time-sensitive data belongs in `SessionStart` (which re-runs with `source: "resume"`), not in a `PostToolUse` string that replays stale.
- **`allowedEnvVars` is the credential-leak guard.** "If you reference `$SLACK_TOKEN` in your headers without listing it in `allowedEnvVars`, Claude Code silently replaces it with the empty string" — frustrating once, prevents a leak the second time.
- **Shell-profile pollution.** An unconditional `echo` in `~/.zshrc` lands in your hook's stdout and breaks JSON parsing; wrap interactive output in `[[ $- == *i* ]]`.
- **Matchers aren't regex unless they contain metacharacters.** `mcp__memory` matches nothing; you need `mcp__memory__.*`. The cleaner v2.1.85 alternative is the `if` field with permission-rule syntax (`"if": "Bash(git *)"`). See [hook matchers](./hook-matchers.md).
- **macOS notifications silently fail** without Script-Editor notification permission — `osascript` "doesn't tell you this is missing."

## Hooks are becoming a category

Portability note worth knowing before you commit: **Codex CLI** ships "almost a direct port" — same JSON-on-stdin protocol, exit codes, and `hookSpecificOutput` shape, but a smaller six-event lifecycle and `command`-only handlers (`prompt`/`agent` are parsed and silently skipped). **OpenCode** is plugin-based (TypeScript modules, block by throwing), and notably **Stop-hook forced verification does not port** because OpenCode can't force the agent to keep working after it's idle. Practical hedge: "write your hooks as standalone shell scripts in a `.claude/hooks/` folder, not as inline command strings" so the logic moves with config translation only.

## Related concepts

- [Hook Lifecycle Events](./hook-lifecycle-events.md) — the events each ladder stage attaches to
- [Hook Types](./hook-types.md) — `command`/`prompt`/`agent` handlers and the `async` flag behind the latency and escalation lessons
- [Hook Decision Control](./hook-decision-control.md) — the `permissionDecision: deny` / `decision: block` mechanics behind Safe YOLO and Stage 4
- [Hook Matchers](./hook-matchers.md) — the matcher-vs-`if` distinction in the gotchas
- [Hook Exit Codes and Output Format](./hook-exit-codes.md) — exit 0 / exit 2 / structured-JSON the scripts rely on
- [Hook Automation Use Cases](./hook-automation-use-cases.md) — generic recipes this playbook prioritizes and orders
- [Production Hook Patterns (ClaudeKit Case Study)](./production-hook-patterns.md) — a shipping kit's design choices; complementary real-world counterpart
- [Permission Modes](./permission-modes.md) — the built-in bypass-mode circuit breakers Safe YOLO extends with custom deny-hooks
- [Hook Scope and Configuration Location](./hook-scope.md) — why the Stage-2 guard belongs in user-global settings
- [StatusLine-Driven Context Backup](./statusline-context-backup.md) — the same source's proactive context-recovery system (the 33K autocompact-buffer math and dual trigger design)
- [The Agent Manager Role](./agent-manager-role.md) — who owns this ladder org-wide once a whole team is on the harness
- [Session Accumulation Pattern](./session-accumulation-pattern.md) — the positive pattern resolving "never put tsc in PostToolUse": journal deltas in PostToolUse, batch-process at Stop
