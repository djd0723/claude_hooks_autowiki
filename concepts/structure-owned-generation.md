---
type: concept
title: "Structure-Owned Generation (Claudux's Deterministic Docs Mode)"
created: 2026-06-29
updated: 2026-06-29
tags: [determinism, docs-generation, claudux, manifest, bounded-patches, guardrails, claude-code]
source_count: 1
sources:
  - sources/clean/github-com-firstbitelabsllc-claudux.md
---

# Structure-Owned Generation (Claudux's Deterministic Docs Mode)

[Claudux](https://github.com/firstbitelabsllc/claudux) is "a local CLI for generating and maintaining VitePress documentation from a codebase," using Claude by default (or Codex with `CLAUDUX_BACKEND=codex`). Its interesting contribution is not "AI writes your docs" — it is the *discipline it imposes on the model*. The thesis is one line:

> "The model can propose wording. The repository owns structure."

This is the same determinism principle that runs through the rest of this wiki — the harness, not the model, owns the boundaries — applied to documentation generation. The mechanism is **deterministic docs mode**: when a repo checks in a `docs-structure.json` manifest, claudux "treats the docs tree as source-owned state, builds a static analysis index before model invocation, and applies model output through bounded section patches instead of broad rewrites."

## Why broad rewrites are the failure mode

The naïve "regenerate the docs with an LLM" approach hands the whole tree to the model and trusts it not to drift, delete, or hallucinate structure. Claudux's deterministic mode is explicitly "meant for larger repos where the structure of the documentation is part of the product" — exactly the regime the [large-codebase playbook](./large-codebase-playbook.md) describes, where you "do the curation work that an index would have done for you, but at the harness layer instead of the model layer." Here the curation artifact is the manifest.

## The manifest owns structure

`docs-structure.json` "owns page IDs, paths, navigation order, source ownership, deletion policy, and required sections." The model never gets to invent any of those. Two index files make the staleness analysis deterministic *before* the model is invoked:

- `.claudux/index/static-analysis.json` "records sorted, hash-based facts about tracked source files, docs files, package scripts, shell dependencies, headings, links, and manifest ownership."
- `.claudux/index/impacted-docs.json` "maps changed files through manifest `source_patterns` and dependency edges to the docs that may be stale."

So the question "which pages need regenerating?" is answered by a static index, not by asking the model to guess.

## Bounded section patches, not rewrites

The model's output is constrained to a patch contract rather than free-form file writing:

> "The model returns JSON between `CLAUDUX_SECTION_PATCHES_JSON_START` and `CLAUDUX_SECTION_PATCHES_JSON_END`." → "Claudux applies only validated patches to manifest-owned generated sections."

This is the documentation analogue of a [hook's structured decision output](./hook-input-output.md): the model speaks through a narrow, machine-validated channel, and everything outside that channel is rejected by the tool, not trusted.

## Guardrails: pinned, read-only, skip, and deletion guards

Several layers stop the model from silently damaging hand-owned content:

- "Pinned sections, read-only sections, and skip-marker blocks are guarded before and after generation."
- **Content protection** auto-protects sensitive paths — "`notes/`, `private/`, `.git/`, `node_modules/`, `*.env`, `*.key`, and `*.pem`" — the same instinct as a committed [`permissions.deny`](./file-permission-patterns.md) boundary.
- Authors fence specific blocks with skip markers (`<!-- skip -->` … `<!-- /skip -->`, plus language variants like `// skip`, `# skip`). In deterministic mode "skip-marker hashes are captured in the guard snapshot so protected blocks cannot be changed silently during generation."
- **Deletion guards**: "AI cleanup paths and claudux recreate are blocked from deleting manifest-owned docs unless explicitly unlocked with environment overrides."

The guard snapshot (hash before, hash after) is what makes "cannot be changed silently" *enforceable* rather than aspirational — the tool can prove a protected block is byte-identical.

## A no-AI readiness path for CI and agent handoff

Determinism also means parts of the workflow need no model at all. `claudux audit` "provides a no-AI readiness snapshot. It reports project detection, manifest validity, link status, checkpoint freshness, changed files since checkpoint, and uncommitted docs/config changes; `--json` makes the same report machine-readable." Claudux is "command-first: most commands inspect or serve local docs, and only `update`, `template`, or the interactive menu need an AI backend." This is what makes it safe in a CI pipeline or a [team-agent handoff](./agent-manager-role.md) — the readiness check is deterministic and the expensive/non-deterministic model step is opt-in.

## Backend determinism caveats

The pattern's guarantees are bounded by the backend's sandbox. The default Claude backend "uses `FORCE_MODEL=sonnet` unless overridden." The Codex backend "runs through `codex exec --json` with `approval_policy="never"`. Its default sandbox is `danger-full-access`; when claudux enters manifest section-patch mode, it requests a read-only sandbox unless `CODEX_SANDBOX_MODE` is set." The takeaway: the section-patch contract is what lets claudux *downgrade* the backend to read-only — structure-ownership and least-privilege reinforce each other.

## The generalizable pattern

Strip away the docs-specific detail and the reusable shape is:

1. **Check the structure into the repo** as a manifest the model cannot edit.
2. **Build a deterministic index** of facts and impacted targets *before* invoking the model.
3. **Constrain model output to validated, bounded patches** of manifest-owned regions only.
4. **Snapshot-guard** anything hand-owned so silent change is impossible.
5. **Expose a no-AI audit** so CI and other agents can gate on readiness without paying for inference.

It is the documentation-generation instance of the wiki's recurring lesson: when you need a guarantee, encode it in something deterministic the model runs *inside of*, not in the model's good behavior.

## Related concepts

- [Generated-Artifact Freshness](./generated-artifact-freshness.md) — the operational/lifecycle half of this pattern: checkpoint drift detection and the graduated `claudux audit` gates
- [Large-Codebase Playbook](./large-codebase-playbook.md) — "structure of the documentation is part of the product"; harness-layer curation at scale
- [Self-Improving CLAUDE.md](./self-improving-claude-md.md) — the same determinism-over-model-behavior instinct applied to CLAUDE.md upkeep
- [The Agent Manager Role](./agent-manager-role.md) — owns the CI/team-agent handoff that `claudux audit` feeds
- [Hook Input/Output](./hook-input-output.md) — structured, validated model output as the safe channel
- [File Permission Patterns](./file-permission-patterns.md) — committed deny boundaries, the permission-layer analogue of content protection
