---
type: concept
title: "Subagent Invocation"
created: 2026-06-29
updated: 2026-06-29
tags: [subagents, delegation, invocation, background, patterns]
source_count: 1
sources:
  - sources/clean/code-claude-com-docs-en-sub-agents-md.md
---

# Subagent Invocation

A [subagent](./subagents.md) runs either because Claude delegates to it automatically or because you invoke it explicitly.

## Automatic delegation

Claude automatically delegates tasks based on the task description in your request, the `description` field in subagent configurations, and current context. To encourage proactive delegation, include phrases like "use proactively" in your subagent's `description`.

## Invoke subagents explicitly

When automatic delegation isn't enough, three patterns escalate from a one-off suggestion to a session-wide default:

- **Natural language** — name the subagent in your prompt; Claude decides whether to delegate. No special syntax (e.g. *"Use the test-runner subagent to fix failing tests"*).
- **@-mention** — type `@` and pick the subagent from the typeahead. This *guarantees* that specific subagent runs rather than leaving the choice to Claude. Your full message still goes to Claude, which writes the subagent's task prompt; the @-mention controls *which* subagent is invoked, not what prompt it receives. You can also type it manually: `@agent-<name>` for local subagents, or `@agent-` followed by the scoped name for plugin subagents (e.g. `@agent-my-plugin:code-reviewer`).
- **Session-wide** — the whole session uses that subagent's system prompt, tool restrictions, and model via `--agent` or the `agent` setting.

### Scoped names

Subagents from an enabled [plugin](./plugin-components.md) appear in the typeahead under their scoped name, such as `my-plugin:code-reviewer`, or `my-plugin:review:security` when the plugin organizes agents into subfolders (see [Subagent Configuration](./subagent-configuration.md)).

### Run the whole session as a subagent

Pass `--agent <name>` to start a session where the main thread itself takes on that subagent's system prompt, tool restrictions, and model:

```bash
claude --agent code-reviewer
```

The subagent's system prompt **replaces the default Claude Code system prompt entirely**, the same way `--system-prompt` does. `CLAUDE.md` files and project memory still load through the normal message flow. The agent name appears as `@<name>` in the startup header, and the choice persists when you resume the session. For a plugin-provided subagent, pass just the agent name; if multiple plugins provide the same name, pass the scoped name (e.g. `my-plugin:security-reviewer`) to disambiguate. To make it the default for every session in a project, set `"agent": "code-reviewer"` in `.claude/settings.json`. The CLI flag overrides the setting if both are present.

## Foreground or background

- **Foreground subagents** block the main conversation until complete. Permission prompts are passed through to you as they come up.
- **Background subagents** run concurrently while you keep working. As of v2.1.186, when a background subagent reaches a tool call needing permission, the prompt surfaces in your main session and names the subagent asking; approve to continue, or press Esc to deny that one call without stopping the subagent. Before v2.1.186, background subagents auto-denied any tool call that would have prompted.

Claude decides foreground vs background based on the task. You can also ask Claude to "run this in the background" or press **Ctrl+B** to background a running task. Set `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS=1` to disable all background task functionality. When [`CLAUDE_CODE_FORK_SUBAGENT`](./forked-subagents.md) is set to `1`, every subagent spawn runs in the background regardless of the `background` field.

## Common patterns

- **Isolate high-volume operations** — delegate verbose work (running tests, fetching docs, processing logs) so the output stays in the subagent's context and only the relevant summary returns. *"Use a subagent to run the test suite and report only the failing tests with their error messages."*
- **Run parallel research** — spawn multiple subagents for independent investigations; Claude synthesizes the findings. Works best when the research paths don't depend on each other. Running many subagents that each return detailed results can itself consume significant context; for sustained parallelism, use agent teams.
- **Chain subagents** — for multi-step workflows, use subagents in sequence; each returns results to Claude, which passes relevant context to the next. *"Use the code-reviewer subagent to find performance issues, then use the optimizer subagent to fix them."*

## Related concepts

- [Subagents](./subagents.md) — the core concept
- [Subagent Configuration](./subagent-configuration.md) — the `agent` setting and scoped identifiers
- [Subagent Context](./subagent-context.md) — why depth is independent of foreground/background
- [Forked Subagents](./forked-subagents.md) — `CLAUDE_CODE_FORK_SUBAGENT` forces all spawns to the background
