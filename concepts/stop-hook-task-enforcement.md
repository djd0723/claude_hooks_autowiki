---
type: concept
title: "Stop-Hook Task Enforcement (Guaranteed Completion)"
created: 2026-06-29
updated: 2026-06-29
tags: [hooks, stop, task-enforcement, test-gate, recursion-guard, claudefast, ralph-wilgum]
source_count: 1
sources:
  - ../sources/clean/claudefa-st-blog-tools-hooks-stop-hook-task-enforcement.md
---

# Stop-Hook Task Enforcement (Guaranteed Completion)

The [`Stop`](./hook-lifecycle-events.md) hook fires when Claude finishes a response. By returning a `block` decision it can refuse to let Claude stop — turning "best effort" into enforced completion. This claudefa.st pattern uses that to gate stopping on objective checks (tests, build, lint) or an explicit task marker, so Claude *literally cannot stop* until the work is verifiably done.

The motivating failure mode:

> "Claude finishes responding but the task is incomplete. Tests are failing. Files are half-written. You ask "are you done?" and Claude says yes, but the build is broken."

## How it works

> "The Stop hook fires every time Claude finishes a response."

From there a Stop hook can:

- **Allow stopping** — exit 0, Claude stops normally.
- **Block stopping** — return `{"decision": "block", "reason": "..."}`, Claude continues working with the reason fed back as the next instruction.
- **Run validations** — execute tests, checks, or validations automatically before either of the above.

The payload is minimal — `session_id`, `transcript_path`, and the critical `stop_hook_active` flag:

```json
{
  "session_id": "uuid-string",
  "stop_hook_active": false,
  "transcript_path": "/path/to/transcript.jsonl"
}
```

## The recursion guard is non-negotiable

A blocking Stop hook can loop forever: Claude responds → hook blocks → Claude continues → responds → hook fires again. The `stop_hook_active` flag is how you break it:

> "The stop_hook_active flag is critical. When true, Claude is already in a "forced continuation" state from a previous block. Always check this to prevent infinite loops."

Every script begins with the guard:

```python
if input_data.get('stop_hook_active', False):
    sys.exit(0)  # Allow stopping, break the loop
```

This is the same recursion guard the [adoption ladder](./hooks-adoption-ladder.md) insists on ("Always check stop_hook_active at the top of the script and exit 0 when it's true") and the same flag the canonical [8-consecutive-block cap](./hook-exit-codes.md) uses as a backstop — after 8 consecutive blocks Claude Code stops calling the hook regardless.

## The enforcement patterns

The source ships four interchangeable gate shapes, all built on the same skeleton (read stdin → guard → run a check → block on non-zero exit):

| Pattern | Check | Block reason on failure |
| :------ | :---- | :---------------------- |
| **Test gate** | `npm test --passWithNoTests` | feeds the last 500 chars of stderr back as context |
| **Build validation** | `npm run build` | "Build failed. Fix compilation errors before completing." |
| **Lint check** | `npx eslint src/ --max-warnings=0` | "Lint errors detected. Run eslint --fix or resolve manually." |
| **Task completion marker** | a `.claude/incomplete-task` file exists | the marker's text: "Task incomplete: {task_info}. Finish it before stopping." |

The test gate's context-passing detail is worth borrowing — instead of a bare "tests failed", it hands Claude the actual output so the continuation is actionable:

```python
stderr = result.stderr.decode()[-500:] if result.stderr else ""
print(json.dumps({"decision": "block", "reason": f"Tests failing. Output: {stderr}"}))
```

Checks chain in a single hook by looping over `(command, error_msg)` pairs and blocking on the first failure (lint → typecheck → test).

## The marker pattern (and "Ralph Wilgum")

The task-completion-marker pattern externalizes "am I done?" into a file Claude must explicitly delete. You write it when work starts and remove it when finished:

```bash
echo "Implement user authentication" > .claude/incomplete-task   # start
rm .claude/incomplete-task                                        # done
```

Generalized, this is the community **"Ralph Wilgum"** pattern for persistent task loops:

> "Create task marker at session start
> Stop hook blocks until marker removed
> Claude must explicitly complete task to remove marker
> No accidental "I'm done" when work is incomplete
> This transforms Claude from "best effort" to "guaranteed completion.""

The Claude Fast Code Kit extends it with a "BiomeValidator Stop hook that runs lint and format checks on every file Claude touches, combined with a build-then-validate pattern where a dedicated quality-engineer agent inspects the builder's output before marking tasks complete" — a [forked subagent](./forked-subagents.md) acting as the verifier, in the same spirit as the [self-improving CLAUDE.md](./self-improving-claude-md.md) Stop-hook reviewer.

## Constraints and gotchas

The Stop hook runs inside a tight, non-interactive envelope. Good and bad fits:

- **Good:** ensuring tests pass before "task complete", validating builds, checking lint/type errors, custom completion criteria.
- **Bad:** long-running operations (60-second hook timeout), network-dependent checks (flaky), blocking on user input (the hook can't interact).

Debugging notes from the source:

- **Infinite loop?** Confirm you're reading `stop_hook_active` correctly; log it (`echo "stop_hook_active: $stop_hook_active" >> ~/.claude/stop-debug.log`).
- **Hook not blocking?** Verify the JSON output is exactly `{"decision": "block", "reason": "..."}`. Per this source, the JSON-decision path uses **exit code 0**, not 2 — its debugging tip is literally "Check exit code is 0 (not 2 for blocking)." (Exit code 2 is the *separate* stderr-based blocking mechanism in [hook exit codes](./hook-exit-codes.md); don't mix the two.)
- **Tests timing out?** The 60-second timeout means you must run a subset or speed the suite up.

A parallelism note that changes the mental model: "Multiple hooks run in parallel. If any returns block, Claude continues." So multiple independent Stop gates compose — any one can keep the turn alive.

## Related concepts

- [Hook Lifecycle Events](./hook-lifecycle-events.md) — where `Stop` sits among the events
- [Hook Decision Control](./hook-decision-control.md) — `decision: block` vs. `additionalContext` for continuing from a Stop hook, and the shared 8-block cap
- [Hook Exit Codes](./hook-exit-codes.md) — exit-code 2 blocking vs. the JSON-decision path this source uses, and the `stop_hook_active` backstop
- [Hooks Adoption Ladder](./hooks-adoption-ladder.md) — the recursion-guard discipline as a first-class rung
- [Self-Improving CLAUDE.md](./self-improving-claude-md.md) — a sibling Stop-hook pattern that *improves* the harness rather than enforcing against it
- [Production Hook Patterns](./production-hook-patterns.md) — ClaudeKit's BiomeValidator-style Stop validation in a shipping kit
- [Forked Subagents](./forked-subagents.md) — the quality-engineer verifier the Code Kit pairs with this gate
