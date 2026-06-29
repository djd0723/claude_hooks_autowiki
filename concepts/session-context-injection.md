---
type: concept
title: "SessionStart Context Injection (Auto-Load Context Pattern)"
created: 2026-06-29
updated: 2026-06-29
tags: [hooks, session-start, context, env-vars, automation]
source_count: 1
sources:
  - sources/clean/claudefa-st-blog-tools-hooks-session-lifecycle-hooks.md
---

# SessionStart Context Injection (Auto-Load Context Pattern)

The [lifecycle reference](./hook-lifecycle-events.md) lists `SessionStart` and `SessionEnd` as two rows in the once-per-session cadence. This page captures the *practitioner pattern* one engineer builds on them: make `SessionStart` the place that auto-loads project context on every session so you never re-prime Claude by hand, and make `SessionEnd` the place that cleans up on the way out.

## The problem it solves

> "Every time you start a Claude Code session, you manually remind it about your project state, environment setup, or current tasks. When sessions end, cleanup tasks are forgotten."

`SessionStart` fires when sessions begin or resume — use it for context that should always be present. The "quick win" is a one-line hook that prints git state on every start:

```
{
  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "echo '## Git' && git branch --show-current && git status --short | head -10"
          }
        ]
      }
    ]
  }
}
```

> "Now every session starts with context. Zero manual setup."

For structured context, a command hook can instead emit JSON with `hookSpecificOutput.additionalContext` — the same [input/output contract](./hook-input-output.md) the [setup pattern](./setup-hooks.md) and [skill activation hook](./skill-activation-hook.md) use to brief Claude before the turn begins.

## The four session matchers

`SessionStart` distinguishes *how* the session began, so you can scope context to the situation:

- **startup** — a new session
- **resume** — from `--resume`, `--continue`, or `/resume`
- **clear** — after `/clear`
- **compact** — after compaction

This is the same matcher axis covered generally in [Hook Matchers](./hook-matchers.md), specialized to session origin. A `resume` session already carries history, so you might inject less; a `compact` session has just lost detail, so you might re-inject the essentials.

## Persisting environment variables with CLAUDE_ENV_FILE

`SessionStart` (and `Setup`) can persist session-wide environment variables by appending to the file named in `CLAUDE_ENV_FILE`. The idiom diffs the environment before and after running setup commands and writes only the new exports:

```
ENV_BEFORE=$(export -p | sort)
source ~/.nvm/nvm.sh
nvm use 20
if [ -n "$CLAUDE_ENV_FILE" ]; then
  ENV_AFTER=$(export -p | sort)
  comm -13 <(echo "$ENV_BEFORE") <(echo "$ENV_AFTER") >> "$CLAUDE_ENV_FILE"
fi
```

> "Variables written to CLAUDE_ENV_FILE are available in all subsequent bash commands Claude runs."

This is the mechanism that lets a hook fix up a tool-version manager (`nvm`, `pyenv`) once at session start instead of in every command — see [Bash Execution Model](./bash-execution-model.md) for why each command otherwise runs in a fresh shell that forgets such changes.

## SessionEnd: cleanup, not blocking

`SessionEnd` fires when a session terminates. It **cannot** block termination, but it can log or clean up. Its input carries a `reason` field:

- **clear** — user ran `/clear`
- **logout** — user logged out
- **prompt_input_exit** — user exited while the prompt was visible
- **other** — other exit reasons

A typical use is appending a row to `session-history.jsonl` for later analysis.

## Where this sits among the lifecycle hooks

Three sibling once-per-session / compaction patterns divide the work:

- **`Setup`** — one-time operations gated behind `--init` / `--maintenance` (install deps, migrations). The decision rule: load git status and inject context on `SessionStart`; install dependencies and run migrations in `Setup`. Captured in [Setup Hooks](./setup-hooks.md).
- **`PreCompact`** — back up the transcript before the window collapses; one practitioner argues for front-running it from the StatusLine instead, in [StatusLine-Driven Context Backup](./statusline-context-backup.md).
- **`SessionStart` / `SessionEnd`** — this page: bracket the session with auto-loaded context and deterministic cleanup.

## Related concepts

- [Hook Lifecycle Events](./hook-lifecycle-events.md) — the `SessionStart`/`SessionEnd` rows and the once-per-session cadence this pattern lives in
- [Setup Hooks (Onboarding & Maintenance Pattern)](./setup-hooks.md) — the sibling `Setup` event, for work you *don't* want on every session
- [StatusLine-Driven Context Backup](./statusline-context-backup.md) — the `PreCompact` backup pattern that brackets the other end of context loss
- [Bash Execution Model](./bash-execution-model.md) — why `CLAUDE_ENV_FILE` is needed for env changes to survive across commands
- [Hook Input and Output](./hook-input-output.md) — the `additionalContext` JSON contract used to inject structured context
- [Hook Matchers](./hook-matchers.md) — the general matcher mechanism the four session matchers specialize
