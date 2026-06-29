---
type: comparison
title: "Deterministic drift detection vs. LLM-in-the-loop drift audit"
created: 2026-06-29
updated: 2026-06-29
tags: [determinism, drift-detection, staleness, claudux, claude-md, ci, judgment-vs-mechanical]
sources:
  - sources/clean/firstbitelabsllc-github-io-claudux-guide-commands.md
  - sources/clean/claudefa-st-blog-tools-hooks-self-improving-claude-md.md
---

# Deterministic drift detection vs. LLM-in-the-loop drift audit

Two of this wiki's practitioner patterns answer the *same* question — *has this generated or
authored artifact drifted out of true with the code it describes?* — and they answer it from
opposite ends of the [deterministic-vs-judgment axis](../synthesis.md) that runs through the
whole corpus.

[Generated-artifact freshness](../concepts/generated-artifact-freshness.md) (claudux) answers it
**mechanically**: a checkpoint SHA in a committed state file, a `git diff` against it, and a
no-AI readiness report. No model is ever asked whether the artifact is stale.

[The self-improving CLAUDE.md pattern](../concepts/self-improving-claude-md.md) (claudefa.st)
answers it with **judgment**: a [`Stop`](../concepts/hook-lifecycle-events.md) hook spawns a
fresh headless `claude -p` session to read the diff *and* the rule files and propose concrete
edits where the prose no longer matches reality.

The difference is not "which is better." It's that the two detect *different kinds of drift* and
cost wildly different amounts — and the right architecture usually runs the cheap one as a gate
in front of the expensive one.

## The fundamental difference: which files changed vs. what the prose now gets wrong

The deterministic detector knows *that* files moved, never *what they mean*. `claudux diff`:

> "Compares the current HEAD against the checkpoint SHA stored in .claudux-state.json, then adds
> uncommitted documentation/config changes such as docs/\*\*, docs-structure.json, docs-map.md,
> .ai-docs-style.md, and docs-site-plan.json. Lists all modified, added, deleted, staged, or
> untracked files that may need doc updates."

That produces a *candidate set* — "files that may need doc updates" — and stops there. It cannot
tell you that a doc's auth section is now a lie; only that the auth source file changed since the
checkpoint.

The LLM auditor reads both sides and judges the *content*. Its prompt "frames the headless
session as an auditor, not an editor," asking for, per changed area, either:

> "No change needed" if existing conventions still hold, OR
> A concrete proposed edit, with the target file path, the specific lines to change, and a
> one-sentence rationale.

That is the gap a `git diff` can't cross: the auditor reads the refactored auth code *and* the
stale rule and writes "this line is now wrong, change it to X." The deterministic detector would
only have flagged that the file is in the changed set.

## How each one works

| | Deterministic drift detection | LLM-in-the-loop drift audit |
| :-- | :-- | :-- |
| **Source** | [generated-artifact-freshness](../concepts/generated-artifact-freshness.md) (claudux CLI) | [self-improving-claude-md](../concepts/self-improving-claude-md.md) (claudefa.st stop hook) |
| **Mechanism** | `diff(checkpoint SHA, HEAD)` + working-tree delta | headless `claude -p` reads diff + every governing `CLAUDE.md` |
| **What it detects** | *Which* tracked files changed since generation | *Whether the prose is now wrong*, and what it should say |
| **Output** | A candidate file list + a no-AI readiness report | A review file of concrete proposed edits + rationales |
| **Cost** | Free — "no backend is invoked, the audit is cheap, fast, and safe" | "adds tokens to every turn… a real cost on long sessions" |
| **Trust model** | Exact and repeatable; git plumbing can't hallucinate | Probabilistic; can over- or under-flag, needs a recursion guard |
| **Acts when** | On demand / in CI (`claudux status`, `audit`, `diff`) | Automatically, at the end of every turn |

## When the deterministic detector is the right tool

Reach for the checkpoint approach when the question really is *mechanical* — when "which
artifacts might be stale" is a sufficient answer and you need it cheap, exact, and pipeline-safe.

- **It belongs in CI.** Because "no backend is invoked," the readiness report is "cheap, fast,
  and safe to run in a pipeline or before handing work to another agent." It speaks in standard
  exit codes, so any build can branch on it (`claudux update || exit 1`).
- **It graduates to the moment.** The same report hardens from advisory to hard failure via
  flags — `--strict` (manifest/link integrity), `--release` (also docs drift), `--handoff-strict`
  (also checkpoint freshness mandatory). An LLM audit has no equivalent: a model's "looks fine"
  is not a gate you can fail a release on.
- **It can't be wrong about the facts.** "The staleness question is answered by Git plumbing and
  a state file — not by asking an LLM 'are these docs out of date?'" There is nothing to verify.

Its ceiling is exactly its floor: it never reads the prose, so it cannot distinguish a
file that changed cosmetically from one whose change invalidated a documented rule. It flags the
suspects; it can't convict.

## When the LLM audit is the right tool

Reach for the headless-audit approach when *the drift is semantic* — when knowing a file changed
tells you nothing useful and you need a judgment about whether a specific rule still holds.

- **The artifact is unstructured prose with no manifest.** A `CLAUDE.md` has no
  `docs-structure.json` pinning sections to source files, so there is no deterministic map from
  "auth.py changed" to "this paragraph is stale." Only a reader of both can tell.
- **You want a proposed *fix*, not just a flag.** The auditor's output is an actionable edit with
  "the specific lines to change, and a one-sentence rationale" — the next step the deterministic
  detector leaves entirely to you.
- **The drift is invisible by construction.** CLAUDE.md "still loads every session. Claude still
  treats every line as authoritative. The rules just happen to describe a codebase that no longer
  exists." Nothing errors, so a mechanical check has nothing to trip on — you need something that
  reads meaning.

Its costs are the inverse of the detector's virtues: tokens on every turn, the risk that the
"reviewer proposes too much," and the need for a recursion guard so you don't get "stop hooks
proposing CLAUDE.md edits about stop hooks proposing CLAUDE.md edits." It is "overkill when you
are a solo developer working on a stable side project."

## They compose: cheap gate in front of expensive judgment

The two are not rivals — they are the two halves of one honest-artifact pipeline, and they stack
in the obvious order. The deterministic detector is the cheap filter that says *which* files are
even in play; the LLM audit is the expensive judgment you spend only on that narrowed set. This
is exactly the [structure-owned generation](../concepts/structure-owned-generation.md) instinct —
"the model can propose wording, the repository owns structure" — applied to staleness:
deterministic plumbing owns *what to look at*, and the model is only trusted for *the reading*.

Both even converge on the same high-stakes boundary. The detector makes checkpoint freshness a
*mandatory* gate at agent handoff (`--handoff-strict`, because "an agent that inherits stale docs
inherits a lie"); the audit feeds [the agent manager's review queue](../concepts/agent-manager-role.md)
"instead of letting rules go stale." One is the mechanical precondition, the other the judgment
that keeps the conventions current — the [operational complement](../concepts/generated-artifact-freshness.md)
and the semantic complement of the same goal.

## Decision table

| Situation | Reach for |
| :-------- | :-------- |
| Need a CI gate that can fail a build deterministically | **Deterministic drift detection** |
| Artifact is manifest-backed (docs tree, generated pages) | **Deterministic drift detection** |
| Question is "which files might need attention," cheaply | **Deterministic drift detection** |
| Gating an agent-to-agent handoff on freshness | **Deterministic drift detection** (`--handoff-strict`) |
| Artifact is unstructured prose (a `CLAUDE.md` hierarchy) | **LLM-in-the-loop drift audit** |
| You want a concrete proposed edit, not just a flag | **LLM-in-the-loop drift audit** |
| Drift is semantic and invisible (rules silently wrong) | **LLM-in-the-loop drift audit** |
| Both: filter the candidate set cheaply, then judge only those | **Run the detector as a gate before the audit** |

## Related

- [Generated-Artifact Freshness](../concepts/generated-artifact-freshness.md) — the deterministic, checkpoint-and-exit-code half of this contrast
- [Self-Improving CLAUDE.md](../concepts/self-improving-claude-md.md) — the LLM-in-the-loop, propose-an-edit half of this contrast
- [Structure-Owned Generation](../concepts/structure-owned-generation.md) — the determinism principle ("repo owns structure, model proposes wording") both inherit
- [The Agent Manager Role](../concepts/agent-manager-role.md) — the human owner whose review queue and handoff boundary both patterns ultimately serve
- [Hook Input/Output](../concepts/hook-input-output.md) — the same narrow-validated-channel discipline, one event earlier
