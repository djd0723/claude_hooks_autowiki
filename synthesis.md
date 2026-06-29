---
type: synthesis
created: 2026-06-29
updated: 2026-06-29
tags: [hooks, plugins, skills, subagents, permissions, settings, tools, agent-sdk, extensibility, determinism, lifecycle, progressive-disclosure, configuration-precedence, skill-activation, path-scoping, harness-ownership, context-engineering, scale, docs-generation, drift-detection, incremental-processing, permission-modes, mode-strategy, practitioner-workflow]
source_count: 29
sources:
  - sources/clean/code-claude-com-docs-en-hooks-guide-md.md
  - sources/clean/code-claude-com-docs-en-hooks-md.md
  - sources/clean/code-claude-com-docs-en-plugins-md.md
  - sources/clean/code-claude-com-docs-en-plugins-reference-md.md
  - sources/clean/code-claude-com-docs-en-skills-md.md
  - sources/clean/code-claude-com-docs-en-sub-agents-md.md
  - sources/clean/code-claude-com-docs-en-permissions-md.md
  - sources/clean/code-claude-com-docs-en-settings-md.md
  - sources/clean/code-claude-com-docs-en-tools-reference-md.md
  - sources/clean/code-claude-com-docs-en-agent-sdk-claude-code-features-md.md
  - sources/clean/code-claude-com-docs-en-agent-sdk-hooks-md.md
  - sources/clean/docs-claudekit-cc-docs-engineer-configuration-hooks.md
  - sources/clean/mlops-community-blog-the-complete-guide-to-claude-code-hooks-automating-your-ai-coding-workflow.md
  - sources/clean/thomas-wiegold-com-blog-claude-code-hooks.md
  - sources/clean/claudefa-st-blog-tools-hooks-claude-code-setup-hooks.md
  - sources/clean/claudefa-st-blog-tools-hooks-hooks-guide.md
  - sources/clean/claudefa-st-blog-tools-hooks-permission-hook-guide.md
  - sources/clean/claudefa-st-blog-tools-hooks-skill-activation-hook.md
  - sources/clean/claudefa-st-blog-tools-hooks-session-lifecycle-hooks.md
  - sources/clean/claudefa-st-blog-guide-configuration-basics.md
  - sources/clean/claudefa-st-blog-guide-mechanics-path-scoped-skills.md
  - sources/clean/claudefa-st-blog-guide-mechanics-claude-skills-guide.md
  - sources/clean/claudefa-st-blog-tools-hooks-self-improving-claude-md.md
  - sources/clean/claudefa-st-blog-guide-development-agent-manager-role.md
  - sources/clean/claudefa-st-blog-guide-development-large-codebase-playbook.md
  - sources/clean/github-com-firstbitelabsllc-claudux.md
  - sources/clean/firstbitelabsllc-github-io-claudux-guide-commands.md
  - sources/clean/github-com-dev-gom-claude-code-marketplace-tree-head-plugins-hook-auto-docs.md
  - sources/clean/claudefa-st-blog-guide-development-permission-management.md
---

# Claude Code Extensibility — Synthesis

The running thesis: what the sources, taken together, add up to. Revised by the
synthesizer as the wiki grows — through-lines, where sources agree, and where they
contradict each other. The wiki began as a hooks-and-plugins corpus; it now spans the
whole Claude Code extensibility surface: **hooks, plugins, skills, subagents, permissions,
settings, the tool layer, and the Agent SDK**.

---

## Core thesis (29 sources)

Claude Code is built on one organizing idea: **the language model is the planner, but it is
not the enforcer.** Everything in the extensibility surface exists to give the user
deterministic control over a non-deterministic agent — to decide what Claude *can* do, to
guarantee that certain things *always* happen, and to shape what Claude *knows* — without
relying on the model to choose correctly. The permissions source states the principle in its
purest form:

> "Permission rules are enforced by Claude Code, not by the model. Instructions in your prompt or `CLAUDE.md` shape what Claude tries to do, but they don't change what Claude Code allows."

The hooks source states the same idea from the other direction:

> "They provide deterministic control over Claude Code's behavior, ensuring certain actions always happen rather than relying on the LLM to choose to run them."

From this root, the surface divides into three concerns, and every source slots into one or
more of them:

- **Enforcement (what Claude may do):** the [permission system](concepts/permission-settings.md),
  [hooks](summaries/hooks.md), [sandboxing](concepts/sandbox-settings.md), and the
  [tool layer](summaries/tools-reference.md). These are deterministic gates that sit *around*
  the model.
- **Capability & knowledge (what Claude can do and knows):** [skills](summaries/skills.md),
  [subagents](summaries/sub-agents.md), bundled tools, and `CLAUDE.md`/rules. These *extend*
  the model — adding procedures, delegating to fresh contexts, and injecting context on demand.
- **Packaging & distribution (how all of the above travels):** [plugins](summaries/plugins-reference.md)
  bundle hooks, skills, agents, MCP/LSP servers, and monitors into a versioned, scoped unit;
  [settings](summaries/settings.md) is the hierarchical file mechanism that wires every scope
  together; and the [Agent SDK](summaries/agent-sdk-features.md) re-exposes the *same*
  filesystem features to programmatic agents.

The deepest through-line is that these are not separate products bolted together — they share
one configuration model, one permission substrate, and one trust posture. A plugin's hook is
mechanically identical to a user hook; an SDK agent reads the same `settings.json` as the CLI;
a skill's `allowed-tools` grant flows through the same permission evaluator as a `/permissions`
rule. **Learn the substrate once and it applies everywhere.**

---

## Through-lines confirmed across sources

### 1. The deterministic / judgment-bearing axis runs through the whole surface

The clearest recurring design decision is *where to put the line between a fixed rule and a
model judgment*. It shows up identically in every cluster:

- **Hooks:** deterministic command/HTTP hooks (exit code + stdout decide) vs. `type: prompt`
  / `type: agent` hooks that delegate a decision to a Claude model or subagent.
- **Permissions:** static allow/ask/deny rules (deterministic) vs. `PreToolUse` hooks that can
  add judgment on top — but, critically, **never below**: "Deny and ask rules are evaluated
  regardless of what a PreToolUse hook returns."
- **Skills:** a skill body is loaded deterministically when invoked, but *whether* Claude
  reaches for it is a model judgment shaped by the description
  ([skill-activation.md](concepts/skill-activation.md): "Activation is description-driven,
  not name-driven"). The practitioner sources now supply skills' *deterministic counterpart*
  too: a [skill-activation hook](concepts/skill-activation-hook.md) on `UserPromptSubmit`
  appends "use this skill" to the prompt so "Claude can't forget because it never had to
  remember" — taking the model's judgment out of the loop exactly as a `type: command` hook
  does. Skills thus span the *whole* axis, judgment activation on one end and hook-forced
  activation on the other.
- **Subagents & the AI-permission-reviewer pattern** ([ai-permission-reviewer.md](concepts/ai-permission-reviewer.md))
  push this furthest: a tiered design where deterministic fast-paths handle the common cases
  and an LLM is consulted only for the ambiguous remainder. **The deterministic tiers are the
  economic argument** — they keep the judgment layer's cost and latency off the hot path.

The standing lesson across sources: a judgment layer where a deterministic rule would do adds
latency, cost, and a new failure mode (refusal, timeout); a deterministic rule where judgment
is needed is brittle. The choice is per-decision, and the sources treat it as the user's.

### 2. Configuration is hierarchical, additive, and "most-restrictive-wins"

Every configurable surface — hooks, permissions, plugins, skills, sandbox — shares one
precedence model: scopes layer (`enterprise/managed > user > project > local`), matching entries
from *all* scopes apply together, and conflicts resolve toward the most restrictive outcome.
The settings source makes the additive half explicit:

> "**Array settings merge across scopes.** When the same array-valued setting (such as `sandbox.filesystem.allowWrite` or `permissions.allow`) appears in multiple scopes, the arrays are **concatenated and deduplicated**, not replaced."

And the restrictive half is stated for permissions —

> "Rules are evaluated in order: deny, then ask, then allow. The first match in that order determines the outcome, and rule specificity doesn't change the order."

— for hooks (the most restrictive `PreToolUse` decision wins; precedence `deny > defer > ask > allow`),
and for the SDK (verbatim the same precedence string). Plugin **scope** maps directly onto this:
a `managed`-scope plugin's hooks inherit enterprise precedence and cannot be overridden by
user or project config. The model is the same wherever you look; the cross-cutting comparisons
([plugin-scopes-vs-hook-scopes.md](comparisons/plugin-scopes-vs-hook-scopes.md)) exist to map
the small places the scope vocabularies diverge.

### 3. The permission system is the enforcement substrate everything else routes through

Permissions are not one feature among many — they are the floor. Hooks fire before permission
checks but cannot loosen them; skills' `allowed-tools` pre-approve *through* the evaluator
rather than around it ("It does not restrict which tools are available"); subagents carry their
own tool allowlist/denylist enforced by the same machinery; sandboxing composes underneath. The
invariant the permissions source insists on: **"If a tool is denied at any level, no other level
can allow it."** Deny is absolute and global. This is why the AI-permission-reviewer pattern is a
*hook* and not a replacement for the permission system — it can only narrow, never widen.

### 4. Progressive disclosure is the token-economy through-line

Skills, subagents, and plugins all solve the same scaling problem: how to make a large library
of capabilities available without paying for all of it on every turn. Skills load their body
"only when it's used, so long reference material costs almost nothing until you need it."
Subagents run in "a fresh, isolated context window" so a delegated task's tokens never pollute
the parent. Plugins keep dormant components packaged but unloaded until enabled. The unifying
principle: **descriptions are always-on and cheap; bodies are on-demand and isolated.** The
skills source's evaluation guidance follows from this — "Seeing a skill trigger tells you Claude
found it, not that it did what you intended" — because the always-on description is what governs
*discovery*, separately from whether the on-demand body *works*.

The practitioner sources push the principle one level deeper.
[Path-scoped skills](concepts/path-scoped-skills.md) bind a skill to a directory glob so its
*description never lands in the system prompt* unless the work touches a matching path — "not
just 'load the body when needed' but 'do not list the skill if the work is somewhere else.'"
That makes even the always-on tier conditional, and it surfaces a prior decision the sources
call "the single most common skill mistake": **rules vs. workflows.** A convention that must
hold every time belongs in `CLAUDE.md` (always in context); a multi-step procedure that fires
only on specific tasks belongs in a skill (loaded on demand). Get it backwards and "you pay
twice" — rules buried in a skill are missed on the 80% of tasks where it never activates, and
workflows stuffed into `CLAUDE.md` "burn tokens in every session even when irrelevant."

### 5. The Agent SDK is the same foundation, re-exposed — and that is a footgun as well as a feature

The SDK sources establish that programmatic agents are not a parallel universe:

> "The Agent SDK is built on the same foundation as Claude Code, which means your SDK agents have access to the same filesystem-based features: project instructions (`CLAUDE.md` and rules), skills, hooks, and more."

`settingSources` opts into (or out of) the CLI's filesystem config; omitting it equals
`["user", "project", "local"]`. The same-foundation promise is also the sharpest caveat in the
corpus — multi-tenant isolation is *not* the default, because managed policy, global config, and
auto-memory load **regardless** of `settingSources`:

> "Do not rely on default `query()` options for multi-tenant isolation. ... For multi-tenant deployments, run each tenant in its own filesystem and set `settingSources: []` plus `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1` in `env`."

The SDK also reveals surface that the CLI docs underplay: TypeScript supports extra hook events
(`SessionStart`, `SessionEnd`, `TeammateIdle`, `TaskCompleted`) beyond Python, and async hook
outputs "can't block, modify, or inject context into the operation since the agent has already
moved on" — the same sync/async tension documented for CLI hooks
([sync-vs-async-hooks.md](comparisons/sync-vs-async-hooks.md)).

### 6. Hook mechanics (stable across every source that touches them)

The hook-specific findings the corpus has confirmed repeatedly:

- **Exit codes are the primary channel:** `0` = success; `2` = hard block; other non-zero =
  proceed-but-warn (first stderr line shown). Structured `hookSpecificOutput` JSON carries richer
  decisions. `async: true` runs in background; `asyncRewake: true` also wakes Claude on exit 2.
- **Decision shapes are event-specific:** there is no single `decision` field — `PreToolUse`
  uses `hookSpecificOutput.permissionDecision` (allow/deny/ask/defer), `PermissionRequest` uses
  `decision.behavior`, `PermissionDenied` uses `retry: true`, `MessageDisplay` uses
  `displayContent`, most others use top-level `{"decision": "block", "reason": "..."}`.
- **Matchers vs. `if`:** `matcher` is coarse (tool name, or per-event dimensions like
  `SessionStart` reason); the `if` field (v2.1.85+) filters on tool *arguments*
  (`"if": "Bash(git *)"`) and **fails open** if arguments can't be parsed
  ([matcher-vs-if-field.md](comparisons/matcher-vs-if-field.md)).
- **A universal stdin envelope** (`session_id`, `transcript_path`, `cwd`, `permission_mode`,
  `effort`, `hook_event_name`, plus `agent_id`/`agent_type` inside subagents) reaches every hook
  with no configuration.
- **Monitors are a distinct, one-way mechanism** — persistent background commands that stream
  stdout as notifications, with no exit-code semantics, restricted to interactive sessions
  ([monitors-vs-command-hooks.md](comparisons/monitors-vs-command-hooks.md)).

### 7. Practitioner sources confirm the model and add the "adoption ladder"

The TIP/community sources (ClaudeKit, MLOps Community, Thomas Wiegold, ClaudeFast) do not
contradict the official references — they validate them and supply the missing *how-to-start*
narrative. The recurring practitioner shape is an **adoption ladder**
([hooks-adoption-ladder.md](concepts/hooks-adoption-ladder.md)): begin with a single
deterministic formatter/logger hook, graduate to blocking enforcement (pre-commit gates, the
Stop-hook task-completion pattern), and only then reach for judgment hooks. ClaudeKit's
[production-hook-patterns.md](concepts/production-hook-patterns.md) and ClaudeFast's
[ai-permission-reviewer.md](concepts/ai-permission-reviewer.md) are the two worked case studies
of the top rung — both third-party tools wiring real judgment into `PreToolUse`/`PermissionRequest`.

The ladder's sharpest *anti*-pattern — "never put `tsc --noEmit` in `PostToolUse`; multiply a
10–30s typecheck by ~50 edits and you've added ~25 minutes of wall-clock" — now has its
positive resolution in the community `hook-auto-docs` plugin's
[session accumulation pattern](concepts/session-accumulation-pattern.md): use `PostToolUse` as a
*lightweight change journal* (record which files were touched, in constant time) and let `Stop`
batch-process only the accumulated delta once. This is the determinism root applied to *cost*
rather than enforcement — "journal the path, not the content" keeps the per-edit step under the
latency budget while a single incremental run at session end stays accurate. The same plugin
also models the cleanest use of `SessionStart` for config migration (it fires before any
`PostToolUse`/`Stop` reads the config, even on `--resume`/`/clear`), tying it back to the
[session-context-injection](concepts/session-context-injection.md) matcher axis. The pattern
generalizes to test selection, cache invalidation, and dependency analysis — any "process the
changes, not everything" workflow.

The practitioner lens does the same favor for *permissions* that the adoption ladder does for
hooks: it reframes the modes catalogued in through-line #3 from a security setting into a
**workflow ergonomic.** The [permission-mode-strategies](concepts/permission-mode-strategies.md)
playbook treats permission prompts as a *flow* problem — "Claude Code asking for permission on
every file edit and command kills your flow" — and answers it with a mode-per-phase mapping
turned by **Shift+Tab** (`normal → auto-accept edits → plan mode → normal`) rather than by
answering dialogs one at a time: explore an unfamiliar codebase in `plan`, build in
`acceptEdits`, drop back to `default` for the risky parts, and reserve `dontAsk`/`bypassPermissions`
for automation and isolated environments. This is the **same loosen-as-trust-grows shape** as the
hooks ladder — start tight, widen deliberately — and crucially it does *not* dent the #3 floor:
the source's own pitfalls reinscribe the enforcement invariant ("deny rules always win," "do not
rely on Bash rules to restrict command arguments — use WebFetch domain rules"), so the ergonomic
dial only ever moves *within* the deny-bounded envelope. Where the
[ai-permission-reviewer](concepts/ai-permission-reviewer.md) automates the approve/deny *decision*
with a hook, the mode playbook automates the approve/deny *posture* by the human picking the right
gear for the task — two practitioner answers to the same friction, one judgment-bearing and one
ergonomic, and the source closes the loop by pairing modes with [sandbox settings](concepts/sandbox-settings.md)
for defense-in-depth.

### 8. The newest sources lift the unit of analysis from *the surface* to *the harness as an owned, scale-engineered, drift-prone artifact*

The earliest sources answered "what can you configure?" The latest cluster — claudefa.st's
[agent-manager](concepts/agent-manager-role.md), [large-codebase playbook](concepts/large-codebase-playbook.md),
and [self-improving CLAUDE.md](concepts/self-improving-claude-md.md), plus the claudux
[structure-owned-generation](concepts/structure-owned-generation.md) /
[generated-artifact-freshness](concepts/generated-artifact-freshness.md) pair — answers a
harder one: **once you have configured all of it, how do you keep it working as the team, the
codebase, and the model all change underneath you?** Three new through-lines emerge, all
consistent with the determinism root above:

- **The harness, not the model, is the performance variable — and it must be *engineered* at
  scale.** The playbook's central correction is that the canonical scaling failure "has nothing
  to do with the model. It is about the absence of a harness." Because Claude Code "navigates a
  codebase the way a software engineer would" (traverse, read, grep) rather than indexing it,
  past ~30,000 lines "the agent can no longer hold a useful mental map in its head," and the fix
  is **context engineering**: "do the curation work that an index would have done for you, but at
  the harness layer instead of the model layer." Its eight strategies are all the extensibility
  surface re-applied as scale tactics — layered `CLAUDE.md`, subdirectory init, codebase maps,
  committed `permissions.deny`/`.ignore`, scoped commands, an LSP MCP for symbol search,
  exploration subagents, and [path-scoped skills](concepts/path-scoped-skills.md) — confirming
  that "learn the substrate once" pays off precisely when the repo gets large.

- **An unowned harness rots; therefore it needs an owner and a review cadence.** The
  [agent-manager role](concepts/agent-manager-role.md) names the owner — a hybrid PM/engineer (or
  minimum-viable single DRI) accountable for the canonical `CLAUDE.md`, the org `settings.json`,
  the approved-skills list, the plugin marketplace, and permissions policy — because without one,
  "the harness becomes the bottleneck instead of the leverage point." Both sources independently
  converge on the same maintenance discipline: "Harnesses rot," so a "meaningful configuration
  review every three to six months, and also after major model releases," because a rule written
  to constrain a weaker model "becomes friction" on a stronger one. This generalizes the
  configuration-precedence through-line into a *temporal* dimension the official docs omit:
  config is not only layered across scopes, it decays over time and must be pruned.

- **Determinism extends past tool-use into the lifecycle of generated artifacts.** The claudux
  pair carries the "model proposes, repository enforces" principle into docs generation —
  "The model can propose wording. The repository owns structure." A checked-in `docs-structure.json`
  manifest owns page IDs and ordering; the model speaks only through validated
  `CLAUDUX_SECTION_PATCHES_JSON` patches (the documentation analogue of a hook's
  [structured decision output](concepts/hook-input-output.md)); and a checkpoint SHA in
  `.claudux-state.json` makes "are these docs stale?" a deterministic Git diff rather than a model
  judgment. The sharpest reusable idea is the **graduated no-AI gate**: one `claudux audit` report
  hardens into a failure at escalating moments — `--strict` (everyday CI) → `--release` (publish) →
  `--handoff-strict` (mandatory checkpoint freshness *before handing work to another agent*). The
  handoff gate is the notable one — "an agent that inherits stale docs inherits a lie" — and it ties
  the determinism root directly to the agent-manager's ownership of agent-to-agent handoffs.

The unifying meta-move: the corpus has graduated from *cataloguing the levers* to *operating the
machine* — who owns it, how it scales, and how it is kept honest over time. The
[self-improving CLAUDE.md hook](concepts/self-improving-claude-md.md) is the emblem of the shift —
"Most hooks enforce something. This one improves something" — a `Stop` hook that audits the rules
against the diff and files proposals to the agent-manager's review queue, closing the loop so the
harness maintains *itself* between human reviews.

---

## Resolved open questions

Earlier syntheses left questions that newer sources have now answered:

- **Async hooks** — fully documented: `async`/`asyncRewake` are first-class, with the explicit
  constraint that async outputs cannot block or inject context after the fact.
- **Real-world hook recipes** — the practitioner sources supply them: pre-commit enforcement,
  Stop-hook task gating, statusline context backup, format-on-write,
  [`SessionStart` context auto-injection](concepts/session-context-injection.md) (with the
  `startup`/`resume`/`clear`/`compact` matcher axis and `CLAUDE_ENV_FILE` for persisting
  session-wide env vars), and the tiered AI permission reviewer. The adoption-ladder framing
  organizes them by risk.
- **Skills vs. CLAUDE.md** — resolved by the skills source: use a skill "when a section of
  CLAUDE.md has grown into a procedure rather than a fact," because skill bodies load lazily.
  The path-scoped-skills source sharpens this into a decision tree —
  **rules** (every-time conventions, always in context) go in `CLAUDE.md`; **workflows**
  (task-specific procedures, loaded on demand) go in a skill; when you need both, split them and
  have the skill reference the rule.
- **Skill discovery reliability** — answered by [skill-activation.md](concepts/skill-activation.md)
  (the claude-skills-guide source): the `description` *is* an intent classifier — "Write it for
  that job. Mention the trigger phrases users actually type. ... Skip marketing language."
  Disambiguation breaks ties toward the *more specific* description, so vague ones ("helps with
  documents") lose to anything sharper. The source also names the recurring failure modes and
  their fixes: the **Silent No-Op** (missing `allowed-tools` or no concrete target — declare the
  tools and reference a real artifact), the **Wrong Trigger** (too-broad description — tighten it
  and add an explicit "out of scope" boundary), and the **Drift** (a skill that breaks when the
  underlying tool changes — version the `SKILL.md` and review it on a cadence).
- **Custom commands** — resolved: "Custom commands have been merged into skills." A
  `commands/deploy.md` and a `skills/deploy/SKILL.md` both create `/deploy`.
- **Plugin layout pitfalls** — resolved: only `plugin.json` lives in `.claude-plugin/`; all other
  component directories sit at the plugin root.
- **SDK feature parity** — resolved: project instructions, skills, and hooks all load from the
  filesystem via `settingSources`, with the multi-tenant isolation caveat above.
- **Subagent economics (partially answered)** — the [large-codebase playbook](concepts/large-codebase-playbook.md)
  supplies a *qualitative* crossover the earlier sources lacked: delegate when the task "eats
  context" while the parent's work "needs context" — the canonical case being **explore vs. edit**
  ("the two activities you should never combine"), where a read-only exploration subagent reads ~50
  files and returns a ~600-word report so the parent keeps its window for editing. At scale this is
  "the trade worth making every time." Still unquantified: the line-count or task-size threshold
  below which the fresh-system-prompt overhead isn't worth it.
- **Who owns the configuration at team scale (newly answered)** — the
  [agent-manager role](concepts/agent-manager-role.md) names the owner (a hybrid PM/engineer, or
  minimum-viable single DRI), the five areas owned (plugin marketplace, approved skills, CLAUDE.md
  conventions, permissions policy, review cadence), and the rollout shape (audit → v1 harness →
  quiet single-team rollout). The unowned-harness failure modes — tribal knowledge, inconsistent
  results, security drift — are the motivation.

---

## Open questions for future sources to resolve

1. **Cross-surface interaction at scale:** when a plugin contributes hooks *and* skills *and*
   agents *and* an MCP server simultaneously, what coordination/ordering problems emerge that
   the per-component docs don't anticipate?
2. **`type: agent` hooks in practice:** the reference allots a 60s timeout / 50 tool-turns. Do
   real deployments find this sufficient for meaningful verification, or too tight?
3. **Subagent economics:** isolated contexts cost a fresh system prompt and lose parent context.
   When is delegation a net win vs. just doing the work inline? No source quantifies the
   crossover.
4. **Monitors beyond log-tailing:** the component is new (v2.1.105+). Do community sources show
   CI/CD bridges, webhook consumers, or file watchers — the practical ceiling of the feature?
5. **Permission rule authoring at team scale:** how do organizations manage large
   allow/ask/deny rule sets across managed + project + user scopes without rule sprawl or
   accidental shadowing?
6. **Agent teams (CLI-only):** the SDK source mentions agent teams that "share a task list and
   message each other directly" as distinct from subagents. The corpus has no dedicated source on
   teams — a gap.

---

## Concept index (pages so far)

**Hooks**
- [Hook Lifecycle Events](concepts/hook-lifecycle-events.md) · [Hook Types](concepts/hook-types.md) · [Hook Matchers](concepts/hook-matchers.md) · [Hook Exit Codes](concepts/hook-exit-codes.md) · [Hook Scope](concepts/hook-scope.md) · [Hook Input/Output](concepts/hook-input-output.md) · [Hook Decision Control](concepts/hook-decision-control.md)
- Practitioner: [Setup Hooks](concepts/setup-hooks.md) · [Hook Automation Use Cases](concepts/hook-automation-use-cases.md) · [Hooks Adoption Ladder](concepts/hooks-adoption-ladder.md) · [Production Hook Patterns](concepts/production-hook-patterns.md) · [AI Permission Reviewer](concepts/ai-permission-reviewer.md) · [Statusline Context Backup](concepts/statusline-context-backup.md) · [SessionStart Context Injection](concepts/session-context-injection.md) · [Skill Activation Hook](concepts/skill-activation-hook.md) · [Self-Improving CLAUDE.md](concepts/self-improving-claude-md.md) · [Session Accumulation Pattern](concepts/session-accumulation-pattern.md)
- Summaries: [Hooks Guide](summaries/hooks-guide.md) · [Hooks Reference](summaries/hooks.md)

**Plugins**
- [Plugin Components](concepts/plugin-components.md) · [Manifest Schema](concepts/plugin-manifest-schema.md) · [Installation Scopes](concepts/plugin-installation-scopes.md) · [Directory Structure](concepts/plugin-directory-structure.md) · [Environment Variables](concepts/plugin-environment-variables.md) · [CLI Commands](concepts/plugin-cli-commands.md) · [Versioning](concepts/plugin-versioning.md) · [Distribution](concepts/plugin-distribution.md) · [Migration](concepts/plugin-migration.md) · [Default Settings](concepts/plugin-default-settings.md) · [Development Workflow](concepts/plugin-development-workflow.md) · [Creating Plugins](concepts/creating-plugins.md) · [Plugins vs Standalone](concepts/plugins-vs-standalone.md)
- Summaries: [Plugins Reference](summaries/plugins-reference.md) · [Plugins Guide](summaries/plugins.md)

**Skills**
- [Skills](concepts/skills.md) · [Bundled Skills](concepts/bundled-skills.md) · [Frontmatter](concepts/skill-frontmatter.md) · [Discovery](concepts/skill-discovery.md) · [Activation](concepts/skill-activation.md) · [Path-Scoped Skills](concepts/path-scoped-skills.md) · [Invocation Control](concepts/skill-invocation-control.md) · [Arguments](concepts/skill-arguments.md) · [Content Lifecycle](concepts/skill-content-lifecycle.md) · [Dynamic Context](concepts/skill-dynamic-context.md) · [Subagent Execution](concepts/skill-subagent-execution.md) · [Evaluation](concepts/skill-evaluation.md)
- Summary: [Skills](summaries/skills.md)

**Subagents**
- [Subagents](concepts/subagents.md) · [Configuration](concepts/subagent-configuration.md) · [Tool Access](concepts/subagent-tool-access.md) · [Context](concepts/subagent-context.md) · [Invocation](concepts/subagent-invocation.md) · [Hooks](concepts/subagent-hooks.md) · [Forked Subagents](concepts/forked-subagents.md)
- Summary: [Sub-agents](summaries/sub-agents.md)

**Permissions & Settings**
- [Permission Settings](concepts/permission-settings.md) · [Evaluation](concepts/permission-evaluation.md) · [Modes](concepts/permission-modes.md) · [Tool Permission Rules](concepts/tool-permission-rules.md) · [Bash Permission Matching](concepts/bash-permission-matching.md) · [File Permission Patterns](concepts/file-permission-patterns.md) · [Sandbox Settings](concepts/sandbox-settings.md)
- Practitioner: [Permission Mode Strategies](concepts/permission-mode-strategies.md)
- [Settings Files](concepts/settings-files.md) · [Settings Precedence](concepts/settings-precedence.md) · [Configuration Scopes](concepts/configuration-scopes.md) · [Managed Settings](concepts/managed-settings.md) · [Environment Variables](concepts/environment-variables.md)
- Summaries: [Permissions](summaries/permissions.md) · [Settings](summaries/settings.md)

**Tools**
- [Built-in Tools](concepts/built-in-tools.md) · [File Tool Behavior](concepts/file-tool-behavior.md) · [Bash Execution Model](concepts/bash-execution-model.md) · [Working Directories](concepts/working-directories.md)
- Summary: [Tools Reference](summaries/tools-reference.md)

**Agent SDK**
- [SDK Callback Hooks](concepts/sdk-callback-hooks.md) · [SDK Hook Events](concepts/sdk-hook-events.md) · [SDK Hook Patterns](concepts/sdk-hook-patterns.md) · [SDK Project Instructions](concepts/sdk-project-instructions.md) · [SDK Setting Sources](concepts/sdk-setting-sources.md) · [SDK Skills Loading](concepts/sdk-skills-loading.md)
- Summaries: [Agent SDK Features](summaries/agent-sdk-features.md) · [Agent SDK Hooks](summaries/agent-sdk-hooks.md)

**Harness operations & scale**
- [The Agent Manager Role](concepts/agent-manager-role.md) · [Large-Codebase Playbook](concepts/large-codebase-playbook.md) · [Self-Improving CLAUDE.md](concepts/self-improving-claude-md.md)
- Generated docs (claudux): [Structure-Owned Generation](concepts/structure-owned-generation.md) · [Generated-Artifact Freshness](concepts/generated-artifact-freshness.md)

**Cross-cutting comparisons**
- [matcher vs if](comparisons/matcher-vs-if-field.md) · [sync vs async hooks](comparisons/sync-vs-async-hooks.md) · [monitors vs command hooks](comparisons/monitors-vs-command-hooks.md) · [plugin scopes vs hook scopes](comparisons/plugin-scopes-vs-hook-scopes.md) · [skills-dir vs marketplace plugins](comparisons/skills-dir-plugins-vs-marketplace-plugins.md) · [SDK extension features](comparisons/sdk-extension-features.md)
