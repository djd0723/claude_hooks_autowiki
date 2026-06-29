---
type: concept
title: "Self-Improving CLAUDE.md (Stop-Hook Rule-Audit Pattern)"
created: 2026-06-29
updated: 2026-06-29
tags: [hooks, stop, claude-md, headless, self-improvement, claudefast, recursion-guard]
source_count: 1
sources:
  - sources/clean/claudefa-st-blog-tools-hooks-self-improving-claude-md.md
---

# Self-Improving CLAUDE.md (Stop-Hook Rule-Audit Pattern)

`CLAUDE.md` is unusually exposed to a quiet failure mode: it loads every session and Claude treats every line as authoritative, so once the code drifts away from the rules, "every session quietly reinforces stale rules." This page captures one claudefa.st practitioner pattern that closes the loop automatically — a [`Stop`](./hook-lifecycle-events.md) hook that, at the end of each turn, spawns a *separate* headless Claude session to compare the diff against the relevant `CLAUDE.md` files and propose edits to a review file you act on later.

> "A stop hook can spawn a headless Claude session to review changes and propose rule updates after each turn."

It is a sibling to the other LLM-in-the-loop and deterministic claudefa.st hooks — the [AI permission reviewer](./ai-permission-reviewer.md), the [skill activation hook](./skill-activation-hook.md), and [setup hooks](./setup-hooks.md) — but it is the only one that *improves* the harness rather than *enforces* against it.

## The problem: rules that describe a codebase that no longer exists

The article opens with a concrete drift story — a careful `CLAUDE.md` written three months ago describing auth, env-var locations, and a legacy webhook handler, after which "the team refactored auth, moved the env loader, and replaced the webhook handler with a queue. The CLAUDE.md never changed."

> "That is the self improving claude md problem, and it is mostly invisible. The file still loads every session. Claude still treats every line as authoritative. The rules just happen to describe a codebase that no longer exists."

The drift is invisible precisely *because* the file keeps loading — nothing errors, the rules just silently mislead. Quoting Cole Medin's demo of the "helpline" reference repo: *"as you evolve your codebase, your rules must evolve too; it is really bad when your CLAUDE.md goes stale."*

## How the pattern works

The mechanics are deliberately simple:

1. Claude finishes its turn → the `Stop` hook fires.
2. A small Python script collects the recent git diff and the contents of *every* `CLAUDE.md` that lives above any changed path.
3. It pipes that bundle into a fresh `claude -p` invocation.
4. That headless session does one job: "compare what just happened in the code to what the rules currently claim, and write its findings to a review file."

The crucial design choice is that **the review file is the output, not the trigger** — nothing auto-edits your `CLAUDE.md`:

> "You read the review at your own pace, accept what makes sense, ignore what does not. The hook keeps the conversation flowing while quietly maintaining a paper trail of conventions that have drifted."

## The three pieces

### 1. The `settings.json` entry

The hook attaches to the `Stop` event in `.claude/settings.json`:

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "uv run --directory \"$CLAUDE_PROJECT_DIR\" python .claude/hooks/propose_claude_md.py"
          }
        ]
      }
    ]
  }
}
```

The [`$CLAUDE_PROJECT_DIR`](./environment-variables.md) variable is load-bearing: "Hooks run from an undefined working directory, and the proposer needs to find the right CLAUDE.md hierarchy. If you skip this and the hook can't locate the file, you'll get silent no-ops every turn." This is an ordinary [`command` hook](./hook-types.md).

### 2. The reviewer script

The script "does four things in order: collect the git diff, walk up from each changed file to find every CLAUDE.md that governs it, build a prompt, and pipe it into a headless Claude session." The diff is capped (helpline uses roughly 12,000 characters) "so the headless call stays cheap," then:

```python
subprocess.run(
    ["claude", "-p", "--output-format", "text"],
    input=prompt,
    env={**os.environ, "HELPLINE_AILAYER_REFLECT_LOCK": "1"},
    capture_output=True,
)
```

Two safeguards make this safer than it sounds:

- **Recursion guard via env var.** The lock variable is the fence: "The spawned session sees it, and any nested hook checks for it and exits early. Without that fence, you'd have stop hooks proposing CLAUDE.md edits about stop hooks proposing CLAUDE.md edits."
- **Deterministic fallback.** "If the claude CLI is unavailable for any reason, the script falls back to a deterministic note listing the changed areas, no LLM needed." This is what lets the hook "degrade gracefully instead of crashing your turn" in CI and offline environments.

### 3. The reviewer prompt

The prompt "frames the headless session as an auditor, not an editor." Cole's working version, roughly:

```
You are auditing whether the project's CLAUDE.md files still match
reality after a coding session.

For each changed area, return exactly one of:
  1. "No change needed" if existing conventions still hold, OR
  2. A concrete proposed edit, with the target file path, the
     specific lines to change, and a one-sentence rationale.

Only flag genuine new conventions, gotchas, commands, or constraints
that are missing from CLAUDE.md. Do not suggest stylistic rewrites.
Do not propose edits for code style, only for documented rules that
no longer hold.

Output to .claude/claude-md-review.md.
```

Two design choices "do most of the work":

- The explicit **"No change needed"** option "keeps the reviewer honest, since 'no change' should be the most common result."
- The **ban on stylistic rewrites** "stops the reviewer from inflating review files with cosmetic suggestions you'd never apply."

The result is a high signal-to-noise paper trail: "Most turns produce the first kind of entry. That is the point. The reviewer is a watchdog, not a refactorer."

## When it earns its keep

The headless session "adds tokens to every turn. That is a real cost on long sessions and on Pro plans." The pattern is worth it when:

- The codebase is actively evolving, with new conventions emerging weekly.
- The `CLAUDE.md` hierarchy is layered, so subdirectory files drift independently.
- Multiple developers share the same `CLAUDE.md` and need a neutral auditor.
- The project is past the early prototype phase.

It is "overkill when you are a solo developer working on a stable side project with one short root CLAUDE.md" — there a periodic manual review is plenty.

## Pitfalls

The article names four traps and their fixes:

- **Reviewer proposes too much** → tighten the prompt, lower temperature, narrow what counts as a change.
- **Reviewer runs on every micro-edit** → "Gate the hook behind a diff-size or file-count threshold so it doesn't fire when you fixed a typo." (The same [stateful-threshold discipline](./production-hook-patterns.md) other production hooks use.)
- **Reviewer auto-applies its own diffs** → "Never let it. Queue everything for human review. The whole point is a paper trail."
- **Reviewer reads the entire CLAUDE.md every turn** → "Walk the path hierarchy and read only files above changed paths."

## Why it matters: a hook that improves, not enforces

Within the harness model, "Most hooks enforce something. This one improves something." The article calls it "the highest-leverage hook in the set, because it keeps the rest of the harness from rotting on the shelf." The same shape generalizes — swap the prompt and you can audit skills that no longer describe their domain, over-broad permissions policy, or (by reversing direction) inject [start-hook context](./session-context-injection.md). The principle is meta-level:

> "any rule file that lives next to code should be audited by something that reads both."

## Related concepts

- [Hook Lifecycle Events](./hook-lifecycle-events.md) — the `Stop` event this hook attaches to
- [Hook Types](./hook-types.md) — the `command` handler it uses
- [Environment Variables](./environment-variables.md) — `$CLAUDE_PROJECT_DIR`, which lets the script locate the `CLAUDE.md` hierarchy
- [AI Permission Reviewer](./ai-permission-reviewer.md) — sibling claudefa.st hook that puts an LLM in the *permission* loop
- [Skill Activation Hook](./skill-activation-hook.md) — sibling deterministic claudefa.st hook
- [Setup Hooks](./setup-hooks.md) — sibling claudefa.st practitioner hook
- [Production Hook Patterns](./production-hook-patterns.md) — the threshold/stateful-firing discipline this pattern depends on
- [Session Context Injection](./session-context-injection.md) — the reverse direction (load context at session start) the article cites as a variant
- [The Agent Manager Role](./agent-manager-role.md) — the owner whose review queue this stop hook feeds to keep CLAUDE.md conventions current
- [Structure-Owned Generation](./structure-owned-generation.md) — the same determinism-over-model-behavior instinct applied to docs generation
