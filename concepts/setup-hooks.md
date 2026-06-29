---
type: concept
title: "Setup Hooks (Onboarding & Maintenance Pattern)"
created: 2026-06-29
updated: 2026-06-29
tags: [hooks, setup, onboarding, ci-cd, maintenance]
source_count: 1
sources:
  - sources/clean/claudefa-st-blog-tools-hooks-claude-code-setup-hooks.md
---

# Setup Hooks (Onboarding & Maintenance Pattern)

The [lifecycle reference](./hook-lifecycle-events.md) lists `Setup` as a once-per-session event in one table row. This page captures the *pattern* built on it: a way to make codebase onboarding and maintenance a single repeatable command that combines a deterministic script with optional agentic oversight. It's mined from one practitioner's production setup.

`Setup` hooks run before the session boots:

> "Setup hooks were released in Claude Code on January 25th, 2026. They're a special hook type that runs before your session starts."

The core idea is to stop choosing between brittle scripts, unpredictable agents, and ignored docs:

> "Pure scripts are predictable but brittle... Pure agents are smart but unpredictable... Pure docs are flexible but manual."

> "The trick is combining all three. Your deterministic scripts handle execution. Agents provide oversight. The result: a living document that executes."

## The CLI surface

A setup hook fires first, then Claude (and optionally a prompt) runs:

```
claude --init            # hook runs, then Claude boots
claude --init "/install" # hook runs, then the /install command runs automatically
```

> "The setup hook runs first, then Claude boots up. The hook can install dependencies, initialize databases, and set up your environment. When it finishes, Claude sees the results and knows what happened."

A maintenance variant uses the same mechanism for recurring upkeep (`claude --maintenance "/maintenance"`), and a pipeline-only flag exits without an interactive session:

> "The --init-only flag is specifically for pipelines. It runs the hook and exits cleanly with a return code, no interactive session."

## Three modes of operation

The same script can be run three ways; the only difference is whether an agent supervises and whether it asks questions first.

| Mode | Invocation | Behavior |
| :--- | :--- | :--- |
| **Deterministic** | `claude --init` | Script runs alone — fast, predictable, CI-friendly |
| **Agentic** | `claude --init "/install"` | Script runs, then an agent reads the logs and reports what happened |
| **Interactive** | `claude --init "/install true"` | Script runs, then the agent asks clarifying questions and adapts |

> "The script is the source of truth. Both hooks and prompts execute the same script. The difference is whether an agent supervises, and whether it asks you questions first."

Interactive mode is the onboarding payoff — the agent walks a new engineer through choices ("Fresh database or keep existing data? Full install or minimal?") that a static script can never make:

> "This is something scripts can never do. They run the same way every time. Agents can ask clarifying questions mid-workflow and adapt to the context."

## How it's wired

Setup hooks are configured under the `Setup` event in `.claude/settings.json`, using a `matcher` to distinguish init from maintenance (see [hook matchers](./hook-matchers.md)):

```json
{
  "hooks": {
    "Setup": [
      {
        "matcher": "init",
        "hooks": [
          { "type": "command", "command": ".claude/hooks/setup_init.py", "timeout": 120 }
        ]
      },
      {
        "matcher": "maintenance",
        "hooks": [
          { "type": "command", "command": ".claude/hooks/setup_maintenance.py", "timeout": 60 }
        ]
      }
    ]
  }
}
```

The script does the deterministic work (install deps, init the DB) and then hands Claude a summary via `additionalContext` — the same JSON-output contract every hook uses (see [hook input and output](./hook-input-output.md)):

```python
print(json.dumps({
    "hookSpecificOutput": {
        "hookEventName": "Setup",
        "additionalContext": "Setup complete. Run 'just be' and 'just fe' to start."
    }
}))
```

Note `Setup` only supports `command` and `mcp_tool` handlers — not `http`, `prompt`, or `agent` hooks (see [hook types](./hook-types.md)).

## The slash-command-reads-the-log half

The agentic and interactive modes lean on a paired slash command that consumes what the script logged:

```
## Workflow
1. Run /prime to understand the codebase
2. Read the log file at .claude/hooks/setup.init.log
3. Analyze for successes and failures
4. Write results to app_docs/install_results.md
5. Report to user
```

> "If something fails, the agent has context to diagnose it. You can add common issues and solutions to the prompt, and the agent will follow them automatically."

## A `just` launcher so nobody memorizes flags

The source wraps every invocation in a `justfile` command runner so the team and its agents learn one verb each:

```
just cldi     # Deterministic setup    -> claude --init
just cldii    # Agentic setup           -> claude --init "/install"
just cldit    # Interactive setup       -> claude --init "/install true"
just cldm     # Deterministic maintenance
just cldmm    # Agentic maintenance      -> claude --maintenance "/maintenance"
```

> "You, your team, and your agents don't need to remember flags more than once. They just run just cldii and everything works."

## Why the split matters

The pattern's value is that it preserves determinism where you need it and adds intelligence where you want it:

> "Determinism preserved: The hook runs the same script every time. No LLM variance in execution. Agents only analyze after the deterministic work is done."

> "CI compatible: GitHub Actions can run claude --init-only and get a clean exit code."

> "A living document that executes: Your installation process is now in natural language, embedded in prompts that agents follow. When you need to update something, you update the prompt."

## Related concepts

- [Hook Lifecycle Events](./hook-lifecycle-events.md) — the `Setup` event row and its once-per-session cadence
- [Hook Types](./hook-types.md) — why `Setup` is limited to `command`/`mcp_tool` handlers
- [Hook Matchers](./hook-matchers.md) — the `init` vs `maintenance` matcher split
- [Hook Input and Output](./hook-input-output.md) — the `additionalContext` JSON contract the setup script uses to brief Claude
- [Hooks Adoption Ladder (Practitioner Playbook)](./hooks-adoption-ladder.md) — the same kit's broader case for moving guarantees into deterministic hooks
- [Hook Automation Use Cases](./hook-automation-use-cases.md) — where session-lifecycle automation sits among the generic recipes
