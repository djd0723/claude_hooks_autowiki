---
type: concept
title: "Generated-Artifact Freshness (Checkpoint Drift Detection & Graduated No-AI Gates)"
created: 2026-06-29
updated: 2026-06-29
tags: [determinism, docs-generation, claudux, checkpoint, ci, drift-detection, handoff, claude-code]
source_count: 1
sources:
  - sources/clean/firstbitelabsllc-github-io-claudux-guide-commands.md
---

# Generated-Artifact Freshness (Checkpoint Drift Detection & Graduated No-AI Gates)

A tool that generates documentation from code faces a problem the [structure-owned-generation discipline](./structure-owned-generation.md) doesn't by itself solve: once the docs are generated, *how do you know they're still true?* Code moves; the generated artifact silently rots. [Claudux](./structure-owned-generation.md)'s CLI answers this with a pattern worth lifting out of the docs domain: **store a checkpoint, compute drift against it deterministically, and gate on freshness with no model in the loop.**

The reusable insight is that the staleness question is answered by Git plumbing and a state file — not by asking an LLM "are these docs out of date?"

## The checkpoint: a SHA that pins "last generation"

Claudux records the commit it last generated from in a state file. `claudux diff`:

> "Compares the current HEAD against the checkpoint SHA stored in .claudux-state.json, then adds uncommitted documentation/config changes such as docs/\*\*, docs-structure.json, docs-map.md, .ai-docs-style.md, and docs-site-plan.json. Lists all modified, added, deleted, staged, or untracked files that may need doc updates."

So "what changed since the docs were generated?" becomes a deterministic diff between two SHAs plus the working-tree delta — exactly the kind of fact a [static analysis index](./structure-owned-generation.md) prefers over model judgment. `claudux status` reads the same checkpoint to report freshness:

> - "Last generation timestamp and checkpoint commit SHA"
> - "Commits behind HEAD (if stale)"
> - "Uncommitted documentation/config changes (if the checkpoint commit is otherwise fresh)"

The checkpoint distinguishes two independent kinds of drift: the *source* moved ahead of the checkpoint (commits behind HEAD), and the *docs/config* themselves were hand-edited since generation (uncommitted changes). Both are staleness signals, and neither requires inference to detect.

## The no-AI readiness report

Drift detection feeds a readiness gate that is explicitly model-free. `claudux audit`:

> "Print a no-AI readiness report for humans, CI, and team-agent handoffs."

It reports "project detection, manifest validity, link status, checkpoint freshness, changed files since checkpoint, and uncommitted docs/config changes." Because no backend is invoked, the audit is cheap, fast, and safe to run in a pipeline or before [handing work to another agent](./agent-manager-role.md) — the same property that makes claudux "command-first: most commands inspect or serve local docs, and only `update`, `template`, or the interactive menu need an AI backend."

## Graduated strictness: one report, four lifecycle gates

The most transferable piece is how the *same* readiness report hardens into a *failure* at different lifecycle moments via flags. The reference spells out exactly when each is for:

> "Use --json when another tool needs to parse the report. Use --strict when invalid manifests or broken links should fail the command. Use --release before tagging or publishing so package metadata, packability, and docs/config drift become hard failures. Use --handoff-strict before passing work to another agent when checkpoint freshness should be mandatory."

This is a graduated-gate ladder rather than one binary check:

- **`--strict`** — manifest/link integrity must hold (everyday CI).
- **`--release`** — *also* package metadata, packability, and docs drift become hard failures (publish gate).
- **`--handoff-strict`** — *also* checkpoint freshness is mandatory (agent-to-agent handoff gate).

Each higher gate adds conditions appropriate to a higher-stakes moment. The mandatory-freshness rule at the handoff boundary is the notable one: an agent that inherits stale docs inherits a lie, so the handoff is where "docs match code" stops being advisory and becomes a precondition — the operational complement to the [agent manager's handoff ownership](./agent-manager-role.md).

## Standard exit codes make the gates composable

Gates are only useful if a pipeline can branch on them, so claudux "follows standard Unix exit code conventions":

> - "0: Success"
> - "1: General error"
> - "2: Incorrect usage"
> - "124: Timeout"
> - "130: Interrupted (Ctrl+C)"

with the canonical CI idiom `claudux update || exit 1  # Fail build if docs generation fails`. The generation command also carries its own strict mode — `claudux update --strict` is documented as "Re-prompt (then error) on broken internal links" — so even the AI-driven step degrades to a hard failure rather than shipping a [broken link](./structure-owned-generation.md) ("Validates internal links (external URLs are skipped) to prevent 404s").

## The generalizable pattern

Independent of documentation, the reusable shape for keeping any *generated* artifact honest is:

1. **Stamp a checkpoint** (the source SHA you generated from) into a committed state file.
2. **Compute drift deterministically** as `diff(checkpoint, HEAD)` plus working-tree delta — never ask the model whether its own output is stale.
3. **Expose a no-AI readiness report** so the freshness check is cheap and safe in CI.
4. **Graduate the gate to the moment**: lenient for everyday CI, stricter before release, strictest (freshness mandatory) before handing off to another agent.
5. **Speak in standard exit codes** so any pipeline can branch on the result.

It is the lifecycle complement to [structure-owned generation](./structure-owned-generation.md): structure-ownership keeps each generation *correct*, and checkpoint freshness keeps the *already-generated* artifact from quietly drifting out of true.

## Related concepts

- [Structure-Owned Generation](./structure-owned-generation.md) — the sibling claudux pattern; this page is its operational/lifecycle half
- [The Agent Manager Role](./agent-manager-role.md) — owns the team-agent handoff that `--handoff-strict` gates
- [Large-Codebase Playbook](./large-codebase-playbook.md) — why drift detection matters when "the structure of the documentation is part of the product"
- [Deterministic drift detection vs. LLM-in-the-loop drift audit](../comparisons/deterministic-drift-detection-vs-llm-drift-audit.md) — this page is the deterministic, no-AI side of that contrast
