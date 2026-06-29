---
type: source
source_url: https://thomas-wiegold.com/blog/claude-code-hooks/
title: "Claude Code Hooks: From Linting to Hardened AI Workflows | Thomas Wiegold Blog"
raw_capture: ../raw/thomas-wiegold-com-blog-claude-code-hooks.html
captured: 2026-06-29
---

Thomas Wiegold

AI ConsultingServicesAboutBlog

Get in Touch

← Back to Blog

AI CodingMay 10, 2026·16 min read

# Claude Code Hooks: From Linting to Hardened AI Workflows

Tags:Claude CodeHooksAI CodingDeveloper Tools

A couple of months ago I wrote a piece arguing that CLAUDE.md is helpful but expensive noise. The short version: you can't trust Claude to follow your instructions consistently, and every paragraph you write to plead with it costs you tokens on every turn. People asked the obvious follow-up. If CLAUDE.md isn't reliable, what is?

This is the answer.

Claude Code hooks let you wire deterministic shell commands and LLM evaluators into specific points in the agent's lifecycle. Not suggestions. Guarantees. The thing that's supposed to happen, happens, every time, without the model deciding whether to bother.

I've spent the last few weeks migrating things out of CLAUDE.md and into hooks, and the difference is hard to overstate. Claude went from "fast typist I have to babysit" to "teammate I can leave alone for a couple of hours". This piece walks you through how to get there. It's structured as four stages of adoption, from the 5-line formatter every project should ship today, all the way up to forced verification that genuinely changes how you work with AI.

We'll also cover what translates to Codex and OpenCode at the end, because hooks are slowly becoming a category, not a Claude-specific feature.

## How Claude Code Hooks Actually Work

A hook is a configuration entry that tells Claude Code to run something (a shell command, an HTTP call, an MCP tool, a Claude prompt, or a full subagent) when a specific event fires in its lifecycle.

If you've used git hooks, the mental model is the same. You wire a script to a known checkpoint, and that script runs every time Claude reaches it. The difference is that Claude Code has dozens of these checkpoints, where git gives you a handful, and the script has structured data about what Claude was about to do or just did.

The simplest possible hook looks like this: "before any Bash command, run my script". Your script gets the command Claude wants to execute on stdin as JSON, and decides whether to allow it, modify it, or block it. That's the whole API. Everything else is just variations on which event you're hooking, and what your script does with the JSON it receives.

So the question becomes: which events are available, and what shape is the data?

The lifecycle is bigger than most articles let on. As of v2.1.116 there are 27 events, but for practical purposes they fall into five buckets:

- Session lifecycle: SessionStart, SessionEnd, Setup

- Agentic loop: PreToolUse, PostToolUse, PostToolBatch, Stop, StopFailure

- Permissions: PermissionRequest, PermissionDenied

- Subagents and teammates: SubagentStart, SubagentStop, TeammateIdle, TaskCreated, TaskCompleted

- System events: PreCompact, PostCompact, FileChanged, ConfigChange, Elicitation, and a handful of others

If you're wondering which one to use for a given idea, 90% of the time it's PreToolUse, PostToolUse, or Stop. Worry about the rest later. The full list lives in the official hooks reference if you want it.

There are five handler types: command (shell command, the default), http (POST to a URL), mcp_tool (call a connected MCP server tool), prompt (single-turn LLM evaluator, runs Haiku by default), and agent (full subagent verifier, marked experimental). Most production hooks are command type, and that's what I'll show in examples below.

Hooks live in JSON config files in this priority order: user-global at ~/.claude/settings.json, project-committed at .claude/settings.json, project-gitignored at .claude/settings.local.json, then managed enterprise policy, then plugins. All hooks merge additively. Higher priority doesn't replace lower priority; everything that matches an event runs.

The protocol is simple. Your hook reads JSON from stdin (session ID, tool input, transcript path, and so on), does its thing, and communicates back through:

- Exit code 0: success, optionally with JSON or plain text on stdout

- Exit code 2: blocking error, stderr is fed back to Claude

- Structured JSON like {"hookSpecificOutput": {"permissionDecision": "deny", "permissionDecisionReason": "..."}} for fine-grained control

One concept catches everyone out: matchers. They look like regex but aren't, unless they contain regex metacharacters. mcp__memory matches no tool. You need mcp__memory__.*. The cleaner alternative added in v2.1.85 is the if field, which uses permission-rule syntax: "if": "Bash(git *)" only fires for git commands, "if": "Edit(*.ts)" only for TypeScript edits. Use it.

## Stage 1: Format and Lint on Every Edit

Start here. Every project gets this hook on day one, and you keep it forever.

```
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "jq -r '.tool_input.file_path' | xargs -I{} sh -c 'case \"{}\" in *.ts|*.tsx|*.js|*.jsx|*.json) npx --no-install oxfmt \"{}\" && npx --no-install oxlint --fix \"{}\" ;; esac'",
            "timeout": 30
          }
        ]
      }
    ]
  }
}
```

That's it. After every file edit Claude makes, oxfmt formats and oxlint runs with autofix. Prettier and ESLint work fine here too, but I've moved everything to the oxc tools lately because they're written in Rust and absurdly fast. That matters because hook latency lives in Claude's hot path. A hook that takes 3 seconds adds 3 seconds to every single tool call.

A few details that matter:

Use --no-install so a missing devDep fails fast instead of triggering a mid-session install you didn't ask for.

Anchor scripts to $CLAUDE_PROJECT_DIR when you have anything more complex than this. Quote the path. Spaces happen.

Don't put tsc --noEmit here. It's the most common trap I see. Typecheck takes 10 to 30 seconds on a real codebase. Multiply that by the 50 file edits Claude makes during a feature, and you've added 25 minutes of wall-clock time for type errors that were going to be caught by the Stop hook anyway. We'll get to that in Stage 4.

If you're on Bun like me, swap npx --no-install for bun x. Same semantics, slightly faster cold start, and you stop seeing npm's "found 0 vulnerabilities" output noise in your hook logs. For Go projects, the equivalent is gofmt -w "$(jq -r .tool_input.file_path)" plus optionally goimports. Same shape, different toolchain.

The whole thing is honestly less interesting than the section length suggests. But it's the obvious starting point, and it's also the only hook 90% of developers ever set up. The next three stages are where the actual leverage is.

One unsexy upside that surprised me: PR review feels different. You stop reading auto-formatted diffs, and Claude stops "fixing" formatting that wasn't broken in the first place. Signal-to-noise on AI commits goes up immediately.

## Stage 2: Security and Guardrails

This is the section that justifies hooks not being optional.

In October 2025, GitHub issue #10077 was filed by a developer who watched Claude Code run rm -rf on their home directory. November 2025 brought #12637, where Claude created a literal ~ directory and a later glob expansion took down everything in the user's actual home. Both incidents happened in standard permission mode, not bypass mode. Both are tagged area:security/bug in Anthropic's tracker. They are not theoretical.

The fix is one PreToolUse hook, deployed at user level so it applies to every project:

```
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "INPUT=$(cat); CMD=$(echo \"$INPUT\" | jq -r '.tool_input.command // empty'); if echo \"$CMD\" | grep -qE '\\brm\\b.*-rf|git\\s+push.*--force.*\\b(main|master)\\b|>\\s*\\.env|--no-verify'; then jq -n --arg r \"Blocked: dangerous command pattern\" '{hookSpecificOutput:{hookEventName:\"PreToolUse\",permissionDecision:\"deny\",permissionDecisionReason:$r}}'; fi; exit 0"
          }
        ]
      }
    ]
  }
}
```

Add patterns as you find them. Mine has grown to about 15 entries over time. Things that get caught: rm -rf of any flavor, force-pushes to protected branches, writes to .env files, git commit --no-verify, chmod 777 on anything, and the occasional dd if=/dev/zero from when Claude got creative about "freeing up disk space". (True story, my SSD survived.)

The reason this is so important goes deeper than "Claude makes mistakes". The model genuinely doesn't understand the OS-level consequences of path expansion. When it sees rm -rf $TARGET_DIR and TARGET_DIR=~/proj, the unsetting of that variable somewhere upstream is invisible to the model. The hook sees the expanded form and stops it.

Now the killer feature: a PreToolUse hook returning permissionDecision: "deny" blocks the tool even under --dangerously-skip-permissions.

That sentence is worth re-reading. I've written separately about how dangerous that flag actually is, so I won't rehash it here. The short version: bypass mode disables the interactive prompts and the auto-mode classifier. It does not disable hooks. So you can run Claude in a sandboxed worktree with --dangerously-skip-permissions for genuine YOLO speed, and your destructive-command guard still has the final word. This is the basis of what the community calls "Safe YOLO". I do most of my heavy refactoring this way now. It's fast, and it's hard-to-impossible to nuke your machine with.

A complementary pattern: .env and .git/ write protection. Add to the same hook. Different incident, same logic. You don't want Claude curiously inspecting your AWS credentials, and you definitely don't want it editing .git/objects because it decided the repo state was confusing.

## Stage 3: Logging and Observability

Now we're past safety and into "what is Claude actually doing". Hooks are the only honest answer.

A PostToolUse hook with no matcher, appending each tool call to a JSONL file:

```
{
  "hooks": {
    "PostToolUse": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "jq -c '{ts: now, session: .session_id, tool: .tool_name, input: .tool_input, ok: (.tool_response.is_error | not)}' >> ~/.claude/audit.jsonl",
            "async": true
          }
        ]
      }
    ]
  }
}
```

Five lines, complete audit trail. The async: true means it doesn't block the agent loop, which matters because you'll have hundreds of these per session.

The HTTP hook variant posts to a centralized collector instead. Slack webhook, Datadog, your own logger, whatever. Watch the allowedEnvVars field carefully here. This is the credential-leak guard. If you reference $SLACK_TOKEN in your headers without listing it in allowedEnvVars, Claude Code silently replaces it with the empty string. Frustrating the first time, prevents a bad day the second.

Cost tracking is a nice extension. Parse tool_response, increment a counter in ~/.claude/state/usage.json, optionally return decision: "block" once you cross a threshold. I haven't bothered. My usage is well within limits, but if you're running multiple agentic loops overnight it's an obvious win.

Once you have a JSONL audit trail, the answers it gives you are surprisingly useful. "What did Claude do during last night's overnight refactor?" One jq query. "Did Claude touch the auth module while it was supposed to be working on something else?" One grep. Past me would have read the entire transcript file looking for that. Present me reads three lines of audit log.

The honest takeaway: there's no other tamper-evident way to capture what Claude actually did across long sessions. The transcript file works for one session, but it's local and gets compacted. Hooks are the export layer.

## Stage 4: Forced Verification with Stop Hooks

This is the section that genuinely changed how I work with Claude. If you stop reading after one part, make it this one.

The problem: Claude says "All done!" and the build is broken. You've all seen this. The standard prompt-engineering response is to add stronger language to CLAUDE.md ("YOU MUST RUN THE TESTS BEFORE STOPPING"), and (per my earlier piece) it does not work. Not reliably. Not for long.

The fix is a Stop hook that runs your real verification commands and forces Claude to keep working if anything fails:

```
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": ".claude/hooks/verify.sh",
            "timeout": 180
          }
        ]
      }
    ]
  }
}
```

And the script:

```
#!/bin/bash
INPUT=$(cat)
[ "$(echo "$INPUT" | jq -r '.stop_hook_active')" = "true" ] && exit 0

cd "$CLAUDE_PROJECT_DIR" || exit 0
OUTPUT=$(npx tsc --noEmit 2>&1 && npx vitest run --reporter=basic --no-watch 2>&1)
if [ $? -ne 0 ]; then
  jq -n --arg out "$(echo "$OUTPUT" | tail -50)" \
    '{decision:"block",reason:("Verification failed:\n" + $out)}'
fi
exit 0
```

A few things to call out. The stop_hook_active check at the start is critical. Without it, you get an infinite loop the first time the gate fails and Claude can't immediately fix it. Every developer who tries Stop hooks has done this once. I've done it twice.

The tail -50 is because Claude doesn't need the full test output, just enough to debug. Keeps the context window lean.

--reporter=basic matters for vitest specifically. The default reporter writes interactive escape sequences that pollute the stderr Claude reads back, and sometimes confuses its parsing of the failure output.

For projects where the verification needs more judgment (does the new code actually solve the user's request, not just "compiles and tests pass"), upgrade to a prompt type hook. It spawns a Haiku evaluator that reads the conversation and returns {"ok": true} or {"ok": false, "reason": "..."}. About a tenth of a cent per fire. Honest about subjective things in a way deterministic shell-out can't be.

A concrete example. When I'm having Claude work on something I built, the deterministic gate runs bun test and bun x tsc --noEmit. That catches the mechanical stuff. But it doesn't catch "Claude implemented the wrong scoring algorithm because it misunderstood the spec". For that, the prompt hook re-reads the conversation and the changed files, then asks itself whether the implementation actually matches what was requested. Slower than shell-out, smarter than tests, and dramatically cheaper than me discovering the issue tomorrow morning.

For the heaviest case, where you need to actually run a tool-using subagent rather than just shell out, there's agent type hooks. Still marked experimental in May 2026, 60-second default timeout, up to 50 tool turns. I use them sparingly because the cost is real, but for overnight refactor verification they're earning their keep.

The Stage 4 pattern is the one that makes Claude feel like a teammate. Once it's wired in, you stop reading "I've completed the task" and start trusting it. Or rather, you don't trust it. The hook does. Same outcome, less anxiety.

If this conceptual neighborhood interests you, my Ralph Loop article covers a related pattern: recursive deterministic loops over probabilistic agents.

## Hooks vs Skills, MCP, and Subagents

Hooks aren't always the right answer. Quick decision matrix:

- Hooks: when something must always happen on a lifecycle event. Formatting, blocking, verification, logging, audit.

- Skills: when you're giving Claude how-to knowledge it should consult when relevant. The repository pattern for data access. Your team's commit message conventions. See my Claude Skills guide for the deeper version.

- MCP: when you need external system access. Database queries, APIs, custom tools Claude calls during the task.

- Subagents: when a task needs an isolated context window. Research subagents feeding writer subagents.

- Plugins: when you're packaging hooks, skills, MCP, or subagents for team distribution.

The honest version: sometimes the right answer is CLAUDE.md plus one hook for the invariant that actually matters. Don't over-engineer. Three hooks that always run beat 30 pages of advisory documentation Claude might or might not follow. (Especially when Claude has not, historically, followed the advisory documentation.)

A heuristic I use: if you're typing "Claude must always..." or "Claude should never..." in your CLAUDE.md, that's a hook. If you're typing "When working on X, prefer Y", that's a skill.

## What About Codex and OpenCode?

Hooks are slowly becoming a category, not a Claude-specific feature. Worth knowing what travels.

Codex CLI ships hooks now, and the design is almost a direct port of Claude Code's. Same JSON-on-stdin protocol, same exit codes, same additionalContext shape, same hookSpecificOutput structure. The lifecycle is much smaller (six events: SessionStart, UserPromptSubmit, PreToolUse, PermissionRequest, PostToolUse, Stop), and only command hooks are implemented. Prompt and agent types are parsed and silently skipped. Stage 1 and Stage 2 hooks port across with config translation only. Stage 4 verification works for the deterministic case. The LLM-evaluated version waits until Codex implements prompt hooks.

OpenCode does it differently. Plugin-based, not config-based. You write TypeScript modules in .opencode/plugins/ that export an async function returning event handlers. The closest equivalents to PreToolUse and PostToolUse are tool.execute.before and tool.execute.after, and you block by throwing an Error (no JSON permission decision). About 25 events available, plus a clean session.idle and an experimental compaction hook that lets you replace the compaction prompt entirely.

OpenCode is genuinely lovely if you're already in TypeScript-land. Type-safe Plugin interface, Bun's $ shell helper baked in, npm distribution. The format-and-lint and security guardrail patterns translate directly. Forced verification with Stop hooks does not, because OpenCode doesn't let you force the agent to keep working after it's idle.

Same fundamental insight in all three (deterministic shell-out at lifecycle events beats prompting), three different implementations, and the Claude Code surface is by some distance the most mature.

Practical bottom line for portability: if you're picking up Claude Code hooks because you might switch tools later, write your hooks as standalone shell scripts in a .claude/hooks/ folder, not as inline command strings. The scripts move to Codex with config translation only. They don't move to OpenCode (different runtime, different API), but at least you're rewriting logic, not archaeology.

## Gotchas Worth Knowing Before You Ship

The kind of thing worth a screenshot:

- Stop hook infinite loops. Always check stop_hook_active at the top of the script and exit 0 when it's true. Every developer learns this once.

- PostToolUse can't undo. By the time it fires, the file is already written. Use PreToolUse for prevention, PostToolUse for reaction.

- Last-write-wins on updatedInput. If multiple PreToolUse hooks rewrite the same tool's input, the order is non-deterministic. Don't have two hooks fighting over the same field.

- additionalContext has a 10,000-character cap and stales on resume. Time-sensitive data goes in SessionStart (which re-runs with source: "resume"), not in PostToolUse (which replays the saved string).

- Shell profile pollution. An unconditional echo in ~/.zshrc will end up in your hook's stdout and break JSON parsing. Wrap interactive output in [[ $- == *i* ]] and your hooks will stop mysteriously failing.

- macOS notification permission. osascript -e 'display notification "..."' routes through Script Editor, which needs explicit notification permission in System Settings. Your hook doesn't tell you this is missing. Notifications just silently don't show up, and you wonder why your beautiful Stop alert never fires.

## Where to Start

Add the Stage 1 formatter today. It takes five minutes. Add the Stage 2 Bash guard this week, in your user-global ~/.claude/settings.json so it applies everywhere. Add Stage 3 logging when you start running Claude unattended or shipping its work to teammates. Add Stage 4 verification when you want to trust Claude with longer sessions.

The whole thing is the deterministic layer that makes the rest of your AI coding setup actually reliable. Once it's in place, you start running Claude longer, paying less attention to formatting nitpicks, and reading "I'm done" with the confidence that comes from knowing the gate confirmed it. Worth the fifteen minutes of config.

Subscribe

### Thomas Wiegold

AI Solutions Developer & Full-Stack Engineer with 14+ years of experience building custom AI systems, chatbots, and modern web applications. Based in Sydney, Australia.

### Related Articles

AI Coding

#### CLAUDE.md: Helpful or Just Expensive Noise?

Research shows CLAUDE.md files can hurt more than help. Here's what actually works—and when to skip it entirely.

9 min read

AI Coding

#### Claude Code dangerously-skip-permissions: Why It's Tempting, Why It's Dangerous

dangerously-skip-permissions makes Claude Code autonomous—no more prompt fatigue. But real devs have lost home directories. Here's what you actually need to know.

11 min read

AI Coding

#### I Switched From Claude Code to OpenCode — Here's Why

I tested OpenCode against Claude Code for real development work. Here's what surprised me about the open-source alternative — and what I don't miss.

8 min read

### Ready to Transform Your Business?

Let's discuss how AI solutions and modern web development can help your business grow.

Get in Touch

### Thomas Wiegold

AI Solutions & Web Development specialist helping SMBs transform their operations with intelligent technology.

🇦🇺Based in Sydney

#### Quick Links

- AI Consulting

- Blog

- Contact

#### Contact

contact@thomas-wiegold.com

Sydney, NSW, Australia

#### Connect

Available for new projects

Currently accepting clients

© 2026 Thomas Wiegold. Made within Sydney

Built withReact RouterandTailwind CSS
