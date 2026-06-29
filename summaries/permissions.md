---
type: summary
title: "Permissions — modes, rule evaluation, and settings"
slug: code-claude-com-docs-en-permissions-md
created: 2026-06-29
updated: 2026-06-29
tags: [permissions, modes, rule-evaluation, settings, bash, managed-settings]
sources:
  - sources/clean/code-claude-com-docs-en-permissions-md.md
---

# Summary: Configure permissions

The authoritative reference for how Claude Code decides what the agent may do: the tiered permission system, allow/ask/deny rules and their precedence, [permission modes](../concepts/permission-modes.md), rule syntax per tool, and how rules interact with sandboxing and managed policy.

## Key thesis from source

> "Claude Code supports fine-grained permissions so that you can specify exactly what the agent is allowed to do and what it can't. Permission settings can be checked into version control and distributed to all developers in your organization, as well as customized by individual developers."

A critical framing the doc repeats: enforcement is by the harness, not the model.

> "Permission rules are enforced by Claude Code, not by the model. Instructions in your prompt or `CLAUDE.md` shape what Claude tries to do, but they don't change what Claude Code allows."

## What this source covers

- The tiered [permission system](../concepts/permission-settings.md) and the `/permissions` management UI
- The three rule kinds (allow / ask / deny) and their fixed [evaluation order](../concepts/permission-evaluation.md)
- The six [permission modes](../concepts/permission-modes.md) and the `defaultMode` setting
- [Tool permission rule](../concepts/tool-permission-rules.md) syntax: bare names, specifiers, input-parameter matching, wildcards, tool-name globs
- Tool-specific rules: [Bash](../concepts/bash-permission-matching.md), PowerShell, [Read/Edit file patterns](../concepts/file-permission-patterns.md), WebFetch, MCP, Agent, Cd
- Extending permissions with PreToolUse hooks; working directories
- How permissions interact with [sandboxing](../concepts/sandbox-settings.md)
- [Managed settings](../concepts/managed-settings.md) and settings precedence

## The tiered permission system

> "Claude Code uses a tiered permission system to balance power and safety"

| Tool type | Example | Approval required | "Yes, don't ask again" behavior |
| :-------- | :------ | :---------------- | :------------------------------ |
| Read-only | File reads, Grep | No | N/A |
| Bash commands | Shell execution | Yes | Permanently per project directory and command |
| File modification | Edit/write files | Yes | Until session end |

## Allow / ask / deny rules and precedence

The three rule kinds, viewable and editable via `/permissions`:

- **Allow** rules let Claude Code use the specified tool without manual approval.
- **Ask** rules prompt for confirmation whenever Claude Code tries to use the specified tool.
- **Deny** rules prevent Claude Code from using the specified tool.

> "Rules are evaluated in order: deny, then ask, then allow. The first match in that order determines the outcome, and rule specificity doesn't change the order."

Key consequences of this fixed precedence:

- A broad deny like `Bash(aws *)` blocks calls that also match a narrower allow like `Bash(aws s3 ls)`; a deny rule can't carry allowlist exceptions.
- A matching ask rule prompts even when a more specific allow rule also matches.
- Deny behavior depends on whether the rule names a tool or scopes a pattern: a bare `Bash` removes the tool from Claude's context entirely (Claude never sees it), while `Bash(rm *)` leaves the tool available and blocks only matching calls.

## Permission modes

Set via `defaultMode` in settings files. See [permission modes](../concepts/permission-modes.md).

| Mode | Description |
| :--- | :---------- |
| `default` | Standard behavior: prompts for permission on first use of each tool |
| `acceptEdits` | Automatically accepts file edits and common filesystem commands such as `mkdir`, `touch`, `mv`, and `cp` for paths in the working directory or `additionalDirectories` |
| `plan` | Plan Mode: Claude reads files and runs read-only shell commands to explore but doesn't edit your source files |
| `auto` | Auto-approves tool calls with background safety checks that verify actions align with your request. Currently a research preview |
| `dontAsk` | Auto-denies tools unless pre-approved via `/permissions` or `permissions.allow` rules |
| `bypassPermissions` | Skips permission prompts, except those forced by explicit `ask` rules. Root and home directory removals such as `rm -rf /` also still prompt as a circuit breaker |

`bypassPermissions` and `auto` can be locked off with `permissions.disableBypassPermissionsMode` or `permissions.disableAutoMode` set to `"disable"` in any settings file — most useful in managed settings where they can't be overridden.

## Rule syntax

Rules follow the format `Tool` or `Tool(specifier)`.

| Form | Effect |
| :--- | :----- |
| `Bash`, `Read`, `WebFetch` | Matches all uses of the tool (`Bash(*)` is equivalent to `Bash`) |
| `Bash(npm run build)` | Matches the exact command |
| `Read(./.env)` | Matches reading `.env` in the current directory |
| `WebFetch(domain:example.com)` | Matches fetches to example.com |
| `Tool(param:value)` | Deny/ask only: matches a top-level scalar input parameter |
| `"mcp__*"`, `"*"` (deny/ask) | Tool-name glob matching all MCP tools / every tool |

**Input-parameter matching** (`Tool(param:value)`) works only for deny and ask rules, on direct scalar fields (not nested ones), one parameter per rule, with `*` as a wildcard. Fields that a tool already canonicalizes — `command` (Bash/PowerShell), `file_path` (Read/Edit/Write), `path` (Grep/Glob), `notebook_path`, `url` (WebFetch) — are not matchable this way; e.g. `Bash(command:rm *)` is ignored with a startup warning. Use `Bash(rm *)` instead.

**Tool-name globs** in the deny/ask position must match the full tool name. Allow rules accept tool-name globs only after a literal `mcp__<server>__` prefix (the server segment must be glob-free); unanchored allow globs like `"*"` or `"mcp__*"` are skipped with a warning. Canonical tool names must be used — the transcript label `Stop Task` is the canonical `TaskStop`, and a rule written `Stop Task` doesn't match.

## Tool-specific rules

| Tool | Matching notes |
| :--- | :------------- |
| [Bash](../concepts/bash-permission-matching.md) | `*` wildcards at any position; ` *` enforces a word boundary (`Bash(ls *)` matches `ls -la` not `lsof`); `:*` suffix equals a trailing ` *`. Shell-operator aware (each subcommand must match). Wrappers `timeout`/`time`/`nice`/`nohup`/`stdbuf` and bare `xargs` are stripped; env runners (`devbox run`, `npx`, `docker exec`) and exec wrappers (`watch`, `find -exec`) are not. A built-in read-only command set (`ls`, `cat`, `grep`, `cd`, read-only `git`, etc.) runs without prompting in every mode |
| PowerShell | Same shape as Bash; aliases canonicalized (`gci`/`ls`/`dir` → `Get-ChildItem`); case-insensitive; AST-parsed compound commands |
| [Read / Edit](../concepts/file-permission-patterns.md) | gitignore-style patterns with four anchors: `//path` (filesystem root), `~/path` (home), `/path` (project root), `path`/`./path` (cwd). `Edit` covers all file-editing tools; symlinks check both link and target (allow needs both, deny matches either) |
| WebFetch | `domain:` prefix matched against hostname; case-insensitive; `*.example.com` matches subdomains but not the apex; interior `*` matches only between dots |
| MCP | `mcp__server` matches all of a server's tools; `mcp__server__tool` matches one tool |
| Agent | `Agent(Explore)`, `Agent(Plan)`, `Agent(my-custom-agent)` to gate subagents |
| Cd | Controls the `/cd` command only (not model-invocable); any `Cd` allow rule switches `/cd` to allowlist mode; whole-path matching with `*`/`**` |

The doc warns that Bash patterns constraining arguments (e.g. `Bash(curl http://github.com/ *)`) are fragile; prefer denying network tools plus `WebFetch(domain:...)`, or a PreToolUse hook.

## Extending permissions with hooks

PreToolUse hooks run before the permission prompt and can deny, force a prompt, or skip it. But:

> "Hook decisions don't bypass permission rules. Deny and ask rules are evaluated regardless of what a PreToolUse hook returns, so a matching deny rule blocks the call and a matching ask rule still prompts even when the hook returned `\"allow\"` or `\"ask\"`."

A hook that exits with code 2 stops the call before permission rules are evaluated, taking precedence over allow rules. See [hooks](../summaries/hooks.md) for the hook protocol.

## Working directories

Access extends beyond the launch directory via `--add-dir` (startup), `/add-dir` (session), or `additionalDirectories` in settings. Files in additional directories follow the same permission rules. Crucially, additional directories grant file access, not configuration: directories in `permissions.additionalDirectories` load no `.claude/` config, while `--add-dir`/`/add-dir` load only a limited set (skills, subagents, `enabledPlugins`/`extraKnownMarketplaces`, and CLAUDE.md only under an env flag).

## Interaction with sandboxing

Permissions and [sandboxing](../concepts/sandbox-settings.md) are complementary layers — permissions control which tools/files/domains Claude may use; sandboxing provides OS-level enforcement for the Bash tool and its children. With the default `autoAllowBashIfSandboxed: true`, sandboxed Bash runs without prompting even under a bare `Bash` ask rule, but content-scoped ask rules (`Bash(git push *)`), explicit deny rules, and `rm`/`rmdir` against critical paths still prompt.

## Managed settings and precedence

[Managed settings](../concepts/managed-settings.md) can't be overridden by user or project settings. Several keys are managed-only, including `allowManagedHooksOnly`, `allowManagedPermissionRulesOnly`, `allowManagedMcpServersOnly`, `sandbox.network.allowManagedDomainsOnly`, and `strictPluginOnlyCustomization`.

Settings precedence (highest first):

1. **Managed settings** — can't be overridden, including by command line arguments
2. **Command line arguments** — temporary session overrides
3. **Local project settings** (`.claude/settings.local.json`)
4. **Shared project settings** (`.claude/settings.json`)
5. **User settings** (`~/.claude/settings.json`)

> "If a tool is denied at any level, no other level can allow it."

Deny wins across scopes in both directions, because deny rules from any scope are evaluated before allow rules.

## Relationship to concept pages

This summary is the primary source for:
- [Permission modes](../concepts/permission-modes.md) — the six modes and `defaultMode`
- [Permission evaluation](../concepts/permission-evaluation.md) — deny → ask → allow order and cross-scope precedence
- [Permission settings](../concepts/permission-settings.md) — the tiered system and settings keys
- [Tool permission rules](../concepts/tool-permission-rules.md) — rule syntax, specifiers, parameter and tool-name matching
- [Bash permission matching](../concepts/bash-permission-matching.md) — wildcards, compound commands, wrappers, read-only set
- [File permission patterns](../concepts/file-permission-patterns.md) — gitignore-style Read/Edit anchors and symlink handling
- [Sandbox settings](../concepts/sandbox-settings.md) — how permission rules merge into sandbox boundaries
- [Managed settings](../concepts/managed-settings.md) — managed-only keys and centralized policy
