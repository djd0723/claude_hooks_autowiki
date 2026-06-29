---
type: concept
title: "Claude Code in Large Codebases (The 8-Strategy Playbook)"
created: 2026-06-29
updated: 2026-06-29
tags: [scale, monorepo, context-engineering, harness, claude-md, lsp, claudefast]
source_count: 1
sources:
  - sources/clean/claudefa-st-blog-guide-development-large-codebase-playbook.md
---

# Claude Code in Large Codebases (The 8-Strategy Playbook)

The single most common Claude Code failure mode is an agent that works on a weekend project and falls apart on a 200k-line monorepo — grep-searching "into a wall of irrelevant matches" and burning context before producing anything. The article's central claim is that this is **not** a model problem:

> "This is the most common Claude Code failure mode, and it has nothing to do with the model. It is about the absence of a harness."

It frames eight strategies (adapted from an Anthropic engineering playbook published May 2026) ordered "from 'free wins everyone should do' to 'patterns reserved for serious scale.'" The owner of this whole stack at an org is the [agent manager](./agent-manager-role.md).

## Why the default setup breaks at scale

The key mental-model correction: Claude Code does **not** index your repo the way Sourcegraph or Cursor does.

> Claude Code "navigates a codebase the way a software engineer would: it traverses the file system, reads files, uses grep to find exactly what it needs."

That is a feature on small projects (no stale index, no embedding pipeline) but has a breaking point: "Once your repo crosses roughly 30,000 lines, the agent can no longer hold a useful mental map in its head." The fix is to "do the curation work that an index would have done for you, but at the harness layer instead of the model layer" — what the article calls **context engineering at scale**: "deliberate shaping of what enters the agent's window instead of dumping the whole tree at it."

## The eight strategies

### 1. Lean and layered CLAUDE.md

The most-misused harness component. Teams pile stack, conventions, deploy commands, and anti-patterns into one root [`CLAUDE.md`](./sdk-project-instructions.md) until "by month three the file is 600 lines, loads on every session, and degrades performance." The scaling pattern is **layering**: keep root CLAUDE.md under 100 lines for cross-cutting context, then "drop a focused CLAUDE.md into each major subdirectory." This works because of walk-up loading — Claude "walks the directory tree from wherever you start the session, loading every CLAUDE.md it encounters on the way up," so each session "ends up with exactly the rules that apply to the code you are touching, and nothing else." Litmus test: if a root rule only matters inside `packages/payments/`, move it there.

### 2. Initialize Claude in the right directory

The [session's launch directory](./working-directories.md) defines its default scope. Run `claude` at the repo root and "the agent treats the whole 200k-line tree as fair game"; run it inside `packages/payments/` and reads scope to that subtree "while still walking up to load every parent CLAUDE.md it finds."

> "Claude works best when it's scoped to the part of the codebase that's actually relevant to the task."

Root initialization still suits architectural reviews and cross-package refactors — "the point is to make the choice deliberately."

### 3. Codebase maps that earn their place in context

For repos with dozens of top-level folders, the pattern is "a lightweight markdown file at the repo root listing each top-level folder with a one-line description." Keep it surgical — one line per folder answering "what kind of code lives here?" — because "a map is not a tour." For monorepos with hundreds of folders, **layer the maps** the same way CLAUDE.md layers. The signal you need one: "Claude keeps asking 'where is the auth code?' or grepping into the wrong package."

### 4. Ignore files and permission denies

"Two thirds of the noise Claude wades through in most large repos is not source code at all" — node_modules, generated protobuf TS, build artifacts, vendored libs. Two tools:

- A root **`.ignore` file** tells Claude to skip whole trees during file-system traversal — "an aggressive .gitignore, scoped to 'things Claude should not even consider.'"
- **`permissions.deny`** in `.claude/settings.json` lists commands and paths Claude must refuse. Anthropic's point is to commit it: "Committing permissions.deny rules in .claude/settings.json means the exclusions are version-controlled" — so they "enforce themselves on every contributor, every CI run, every new hire." (The mechanics live in [file-permission patterns](./file-permission-patterns.md) and [permission settings](./permission-settings.md).)

### 5. Scoped commands per subdirectory

If a CLAUDE.md three folders up says `pnpm test` and that runs 4,000 tests across 12 packages, Claude "will dutifully kick that off and then sit for nine minutes." The fix is to define test/lint/build commands in the **local** CLAUDE.md scoped to that subtree (e.g. `pnpm --filter @repo/api test`). It "breaks down in compiled-language monorepos with deep cross-directory dependencies" — there, scope what you can and add an explicit note for when a global build is unavoidable. For Turborepo/Nx/Bazel/Pants, "codify the exact incantation. Otherwise Claude will reinvent it badly on each session."

### 6. LSP for symbol-level search

"At 50,000 lines, grep is fine. At 500,000 lines, grep stops being a tool and becomes a tax." A find-references on `handlePayment` returns 3,000 matches across string literals, comments, and unrelated functions. The Language Server Protocol "knows about scopes, types, imports, and inheritance," so it answers with one definition and the real references. You expose it "through an MCP server that wraps a language server binary (rust-analyzer, tsserver, gopls, clangd, whatever your stack uses)" surfacing `find_definition` / `find_references` / `find_implementations`. Anthropic's example: one enterprise "deployed LSP org-wide before broad rollout of Claude Code specifically for C and C++ navigation reliability." Critical detail teams miss — tell Claude *when* to prefer it: "For symbol lookups, use the LSP MCP. For string literal or comment searches, use grep."

### 7. Sub-agents for exploration vs. editing

> "The two activities you should never combine in a single Claude session are mapping a subsystem and editing it. Exploration eats context. Editing needs context."

The pattern: spin up a read-only [exploration sub-agent](./forked-subagents.md) with [its own fresh context window](./subagent-context.md) that "traverses the relevant code, and produces a focused report." The parent receives only the report (a few hundred tokens) and uses the freed context to edit. "On a 200k-line repo, an exploration sub-agent might read 50 files and produce a 600-word summary." The cost — extra tokens — is "the trade worth making every time" at scale.

### 8. Path-scoped skills

The conventions-vs-workflows split: "A naming rule belongs in CLAUDE.md. The seven-step API endpoint creation process belongs in a skill." [Skills](./skills.md) load on demand by task description, and on monorepos "the leverage gets bigger when you scope skills by file path" via a [`paths:` predicate](./path-scoped-skills.md). A `create-api-endpoint` skill scoped to `packages/api/**` "will not load when you are editing a React component." Together, "path-scoped skills plus layered CLAUDE.md plus codebase maps give you per-directory conventions, per-directory workflows, and per-directory navigation."

## When the playbook breaks down

Honest limits the article names:

- **Sheer scale.** "Codebases with hundreds of thousands of folders and millions of files." Past that, "hierarchical CLAUDE.md loading becomes a session-startup cost in its own right."
- **Non-git version control.** The playbook assumes git; Perforce/Mercurial/Plastic "need custom hooks or have to accept that some defaults will not fire." (Anthropic shipped native Perforce support "only after one customer built a hook that intercepted file writes to enforce `p4 edit`.")
- **Game engines with binary assets.** `.fbx`/`.psd`/`.uasset` repos "throw off Claude's traversal heuristics" — use `.ignore` aggressively and map the binary dirs.
- **Non-engineer contributors.** Designer/PM/marketer folders make structural assumptions wobble; local CLAUDE.md tuned to those conventions helps.

## Languages worth calling out

A counter-intuitive data point: "Claude Code performs better than most teams expect on C, C++, C#, Java, PHP, particularly as of recent model releases" — against the common assumption that AI tools work best on the Python/TypeScript/Go languages over-represented in training data. "The C/C++ case is exactly where LSP shifts from 'nice to have' to 'non-negotiable,' but the underlying model performance is no longer the bottleneck." For Java enterprise monorepos, "combine subdirectory initialization with layered CLAUDE.md and an LSP MCP server."

## The configuration review cadence

"Harnesses rot." A rule written to constrain a weaker model becomes friction on a stronger one. Anthropic's recommendation: a meaningful harness review "every three to six months," and after every major model release. The canonical example: "A CLAUDE.md rule that tells Claude to break every refactor into single-file changes" helped older models but "becomes friction" on a newer one. Review steps to bake in: read every CLAUDE.md end to end ("would I still write this rule today?"); audit `permissions.deny` for obsolete protections; check hooks for behavior now native to the platform; and prune skills "that have not loaded in 90 days." The review can be partially automated with a stop hook — see the [self-improving CLAUDE.md pattern](./self-improving-claude-md.md).

## The pattern worth internalizing

> "Every team that succeeds with Claude Code at scale has invested in their harness. There are no exceptions."

The summary maps each strategy to a faculty it gives the agent: "CLAUDE.md tells Claude the rules. Subdirectory initialization tells it the scope. Codebase maps tell it the geography. Ignore files and permissions tell it the boundaries. Scoped commands tell it the feedback loop. LSP tells it about symbols. Sub-agents give it more context window. Path-scoped skills give it the workflows." None is revolutionary alone; combined they are "the difference between an agent that works on toy projects and an agent that ships features in your monorepo." The closing advice is to pick "the two or three that hurt most on your current repo, and ship them this week."

## Related concepts

- [The Agent Manager Role](./agent-manager-role.md) — who owns this whole stack once a team scales
- [SDK Project Instructions](./sdk-project-instructions.md) — CLAUDE.md, the home of layered rules (Strategy 1)
- [Working Directories](./working-directories.md) — the launch-directory scoping behind Strategy 2
- [Path-Scoped Skills](./path-scoped-skills.md) — the `paths:` predicate of Strategy 8
- [Forked Subagents](./forked-subagents.md) / [Subagent Context](./subagent-context.md) — the exploration-vs-editing split of Strategy 7
- [File Permission Patterns](./file-permission-patterns.md) / [Permission Settings](./permission-settings.md) — the committed `permissions.deny` of Strategy 4
- [Self-Improving CLAUDE.md](./self-improving-claude-md.md) — automating the configuration review cadence
