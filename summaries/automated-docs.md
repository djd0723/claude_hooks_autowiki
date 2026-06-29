---
type: summary
title: "Deterministic docs generation — harness-owned structure and freshness"
created: 2026-06-29
updated: 2026-06-29
tags: [determinism, docs-generation, claudux, drift-detection, bounded-patches, practitioner-workflow]
source_count: 2
sources:
  - sources/clean/github-com-firstbitelabsllc-claudux.md
  - sources/clean/firstbitelabsllc-github-io-claudux-guide-commands.md
---

# Summary: Generating docs without letting the model own structure

What the wiki knows about using an LLM to generate and maintain documentation while keeping the
repository — not the model — in control of structure and freshness. The running example is
[Claudux](https://github.com/firstbitelabsllc/claudux), a CLI that generates VitePress docs from
a codebase, but the patterns generalize.

## The core thesis

> "The model can propose wording. The repository owns structure."

This is the wiki's recurring determinism principle — the harness owns the boundaries, not the
model — applied to documentation generation.

## Key points

- [Structure-owned generation](../concepts/structure-owned-generation.md) — a checked-in
  `docs-structure.json` manifest owns page IDs, paths, nav order, and required sections; the
  model speaks only through validated **bounded section patches**, the documentation analogue of
  a [hook's structured decision output](../concepts/hook-input-output.md).
- [Generated-artifact freshness](../concepts/generated-artifact-freshness.md) — once docs are
  generated, staleness is answered by Git plumbing and a checkpoint SHA, not by asking the model
  "are these out of date?" A no-AI readiness `audit` gates freshness in CI and at
  [agent handoff](../concepts/agent-manager-role.md).

## Related topics

- [Claude Code at scale](large-codebase.md) — the same "curate at the harness layer, not the
  model layer" move, for whole-repo navigation rather than docs.
- [Hooks guide](hooks-guide.md) — structured, machine-validated decision channels in the hook system.
