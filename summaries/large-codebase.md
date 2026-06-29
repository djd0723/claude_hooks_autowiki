---
type: summary
title: "Claude Code at scale — large codebases and harness ownership"
created: 2026-06-29
updated: 2026-06-29
tags: [scale, monorepo, context-engineering, harness, orchestration, practitioner-workflow]
source_count: 2
sources:
  - sources/clean/claudefa-st-blog-guide-development-large-codebase-playbook.md
  - sources/clean/claudefa-st-blog-guide-development-agent-manager-role.md
---

# Summary: Running Claude Code at scale

What the wiki knows about making Claude Code work on large codebases and across teams — the
regime where the default "navigate like an engineer" behaviour breaks down and the work shifts
from prompting the model to engineering the harness around it.

## The core reframing

The most common failure mode — an agent that thrives on a weekend project and falls apart on a
200k-line monorepo — is **not** a model problem:

> "This is the most common Claude Code failure mode, and it has nothing to do with the model. It is about the absence of a harness."

Claude Code doesn't index a repo the way Sourcegraph or Cursor do; it traverses the filesystem
and greps. Past ~30,000 lines the agent "can no longer hold a useful mental map in its head",
so the fix is **context engineering at scale** — doing the curation an index would have done,
but at the harness layer.

## Key points

- [Large-codebase playbook](../concepts/large-codebase-playbook.md) — the eight strategies
  (ordered "free wins" → "serious scale"): lean/layered [CLAUDE.md](../concepts/sdk-project-instructions.md),
  launching in the right [working directory](../concepts/working-directories.md), and more.
- [The agent manager role](../concepts/agent-manager-role.md) — who owns the harness (agent
  definitions, hooks, permission policy) once a team adopts it at scale, and why that ownership
  keeps the setup coherent. The large-codebase playbook names this role as the owner of the whole stack.

## Related topics

- [Subagents](sub-agents.md) — the fleet the manager governs.
- [Deterministic docs generation](automated-docs.md) — the same "harness owns the boundaries,
  not the model" principle applied to generated artifacts.
- [Hooks guide](hooks-guide.md) — the enforcement layer the harness is built from.
