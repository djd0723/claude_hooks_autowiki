---
type: concept
title: "Hook Security"
created: 2026-06-29
updated: 2026-06-29
tags: [hooks, security, shell, hardening, validation]
source_count: 1
sources:
  - sources/clean/code-claude-com-docs-en-hooks-md.md
---

# Hook Security

Hooks are arbitrary code that Claude Code runs automatically on your behalf, so they carry the full risk of whatever they execute. The Hooks reference frames this bluntly:

> Command hooks run with your system user's full permissions.

The accompanying warning spells out the blast radius:

> Command hooks execute shell commands with your full user permissions. They can modify, delete, or access any files your user account can access. Review and test all hook commands before adding them to your configuration.

Because a matching hook fires without a confirmation prompt, a poorly written or untrusted hook is a standing liability — it runs on every qualifying event, not just when you are watching.

## Security best practices

The reference lists the practices to keep in mind when writing hooks:

> * **Validate and sanitize inputs**: never trust input data blindly
> * **Always quote shell variables**: use `"$VAR"` not `$VAR`
> * **Block path traversal**: check for `..` in file paths
> * **Use absolute paths**: specify full paths for scripts. In exec form, use `${CLAUDE_PROJECT_DIR}` and the path needs no quoting. In shell form, wrap it in double quotes
> * **Skip sensitive files**: avoid `.env`, `.git/`, keys, etc.

These collapse into three habits:

- **Treat hook stdin as hostile.** The JSON a hook reads on stdin carries model- and tool-derived fields (file paths, command strings, prompts). Validate and sanitize before acting on any of it.
- **Make the shell safe.** Quote every variable expansion, and prefer absolute script paths over relative ones so a changed working directory cannot redirect execution. `${CLAUDE_PROJECT_DIR}` gives a stable root in exec form without needing quotes (see [Environment Variables](./environment-variables.md)).
- **Scope what a hook can touch.** Reject inputs containing `..`, and exclude secret-bearing paths (`.env`, `.git/`, key material) from anything a hook reads, writes, or deletes.

## Relationship to other guardrails

Hook-level hygiene is one layer; it does not replace Claude Code's permission system or sandbox:

- A `PreToolUse` hook can *deny* a tool call, but that decision is only as trustworthy as the input validation behind it — see [Hook Decision Control](./hook-decision-control.md).
- Permission rules and the sandbox constrain what tools (and therefore hook-triggering actions) may do at all; hook hardening complements them rather than substituting for them — see [Sandbox Settings](./sandbox-settings.md) and [Permission Modes](./permission-modes.md).

## Related concepts

- [Hook Types](./hook-types.md) — command hooks (the kind this guidance governs) versus `http`, `prompt`, `agent`, and `mcp_tool` hooks
- [Hook Input and Output](./hook-input-output.md) — the JSON stdin/stdout protocol whose inputs must be validated
- [Hook Decision Control](./hook-decision-control.md) — turning validated input into allow/deny/ask decisions
- [Hook Scope](./hook-scope.md) — where hooks are configured and which scopes are trusted
- [Environment Variables](./environment-variables.md) — `${CLAUDE_PROJECT_DIR}` and the variables hooks can rely on
- [Hooks Adoption Ladder](./hooks-adoption-ladder.md) — adopting hooks incrementally so each rung stays reviewable
