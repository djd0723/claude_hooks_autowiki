---
type: concept
title: "Settings Precedence"
created: 2026-06-29
updated: 2026-06-29
tags: [settings, precedence, scopes, merge, managed]
source_count: 1
sources:
  - sources/clean/code-claude-com-docs-en-settings-md.md
---

# Settings Precedence

When the same setting appears in more than one [scope](./configuration-scopes.md), Claude Code resolves it by precedence. This guarantees organizational policy is always enforced while still letting teams and individuals customize.

## Precedence order (highest to lowest)

1. **Managed settings** — server-managed, MDM/OS-level policies, or `managed-settings.json`. Cannot be overridden by anything, *including command-line arguments*.
2. **Command-line arguments** — temporary session overrides (`--settings <file-or-json>` merges using the same rules as the file layers).
3. **Local project settings** — `.claude/settings.local.json`.
4. **Shared project settings** — `.claude/settings.json`.
5. **User settings** — `~/.claude/settings.json`.

> For example, if user settings set `permissions.defaultMode` to `acceptEdits` and a project's shared settings set it to `default`, the project value applies.

The same order applies whether you run Claude Code from the CLI, the VS Code extension, or a JetBrains IDE.

### Within the managed tier

Only **one** managed source is used; tiers do not merge across each other. The internal order is:

> server-managed > MDM/OS-level policies > file-based (`managed-settings.d/*.json` + `managed-settings.json`) > HKCU registry (Windows only).

Within the file-based tier, drop-in files and the base file *are* merged together (see [Settings Files](./settings-files.md)).

## Scalars override, arrays merge

The default rule is: **scalar values from a higher-priority scope override; array-valued settings concatenate and de-duplicate across all scopes.**

> If managed settings set `allowWrite` to `["/opt/company-tools"]` and a user adds `["~/.kube"]`, both paths are included in the final configuration.

This is why a lower-priority scope can *add* entries (e.g. a [permission](./permission-settings.md) `allow` rule, or a [`sandbox.filesystem.allowWrite`](./sandbox-settings.md) path) without losing the higher-priority ones — and vice versa.

### Two arrays that do *not* merge

- **`fallbackModel`** — an ordered chain where position carries meaning; the highest-precedence file that defines it supplies the entire value.
- **`availableModels`** — when the highest-precedence managed source defines it, that list applies as-is and lower scopes cannot extend it; across non-managed scopes it merges as usual (v2.1.175+).

## Verifying which sources are active

Run `/status` inside Claude Code. The **Status** tab's `Setting sources` line lists each loaded layer (e.g. `User settings`, `Project local settings`). When managed settings are in effect, the entry shows the delivery channel in parentheses — `(remote)`, `(plist)`, `(HKLM)`, `(HKCU)`, or `(file)`. A layer appears only when it is loaded with at least one key; an empty list means no sources were found. The line confirms *which* sources are read, not which layer supplied each individual key. If a file has errors, `/status` lists it and `/doctor` shows the details.

## Related concepts

- [Configuration Scopes](./configuration-scopes.md) — what each layer is and who it affects
- [Settings Files](./settings-files.md) — the files and managed delivery mechanisms being ranked
- [Permission Settings](./permission-settings.md) — permission rules rely on the array-merge behavior
- [Sandbox Settings](./sandbox-settings.md) — `sandbox.filesystem.*` and `network.*` arrays merge across scopes
