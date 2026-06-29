---
type: concept
title: "Sandbox Settings"
created: 2026-06-29
updated: 2026-06-29
tags: [settings, sandbox, security, filesystem, network]
source_count: 1
sources:
  - sources/clean/code-claude-com-docs-en-settings-md.md
---

# Sandbox Settings

The `sandbox` block in [`settings.json`](./settings-files.md) isolates bash commands from your filesystem and network (macOS, Linux, and WSL2). When sandboxing is on, commands run inside an OS-level boundary unless explicitly excluded.

## Core toggles

| Key | Meaning |
| :-- | :------ |
| `enabled` | Enable bash sandboxing. Default `false` |
| `failIfUnavailable` | Exit at startup if `enabled` is true but the sandbox can't start (missing deps / unsupported platform). Default `false` (warn and run unsandboxed). Intended as a hard gate for managed deployments |
| `autoAllowBashIfSandboxed` | Auto-approve bash commands when sandboxed. Default `true` |
| `excludedCommands` | Commands that run *outside* the sandbox, e.g. `["docker *"]` |
| `allowUnsandboxedCommands` | When `false`, disables the `dangerouslyDisableSandbox` escape hatch so all commands must be sandboxed (or in `excludedCommands`). Default `true` |

## Filesystem restrictions (`sandbox.filesystem`)

These control paths at the OS sandbox boundary and apply to **all** subprocesses (`kubectl`, `terraform`, `npm`), not just Claude's file tools. All arrays **merge across [scopes](./settings-precedence.md)** and also merge with paths from `Edit`/`Read` [permission rules](./permission-settings.md).

| Key | Meaning |
| :-- | :------ |
| `allowWrite` | Additional paths sandboxed commands may write |
| `denyWrite` | Paths sandboxed commands cannot write |
| `denyRead` | Paths sandboxed commands cannot read |
| `allowRead` | Re-allow reads within `denyRead` regions (takes precedence over `denyRead`) |
| `allowManagedReadPathsOnly` | (Managed only) honor only managed `allowRead` paths; `denyRead` still merges |

`credentials.files` and `credentials.envVars` (v2.1.187+) group credential-specific read blocks and env-var scrubbing separately, with `{ "path"/"name": ..., "mode": "deny" }` entries.

### Path prefixes

Filesystem keys use **standard path conventions** (which differ from [`Read`/`Edit` permission rules](./permission-settings.md)):

| Prefix | Meaning |
| :----- | :------ |
| `/` | Absolute path from filesystem root |
| `~/` | Relative to home directory |
| `./` or none | Project root (project settings) or `~/.claude` (user settings) |

The older `//path` form for absolute paths still works; switch single-slash `/path` (if you relied on project-relative resolution) to `./path`.

## Network restrictions (`sandbox.network`)

| Key | Meaning |
| :-- | :------ |
| `allowedDomains` | Outbound domains to allow (wildcards like `*.example.com`) |
| `deniedDomains` | Domains to block; takes precedence over `allowedDomains`; merged from all sources regardless of `allowManagedDomainsOnly` |
| `allowManagedDomainsOnly` | (Managed only) honor only managed `allowedDomains` / `WebFetch(domain:...)` rules |
| `allowUnixSockets` / `allowAllUnixSockets` | Unix socket access (on Linux/WSL2 only `allowAllUnixSockets` works) |
| `allowLocalBinding` | Bind to localhost ports (macOS only) |
| `httpProxyPort` / `socksProxyPort` | Bring-your-own proxy ports |

## Example

```json
{
  "sandbox": {
    "enabled": true,
    "autoAllowBashIfSandboxed": true,
    "excludedCommands": ["docker *"],
    "filesystem": {
      "allowWrite": ["/tmp/build", "~/.kube"],
      "denyRead": ["~/.aws/credentials"]
    },
    "network": {
      "allowedDomains": ["github.com", "*.npmjs.org"],
      "deniedDomains": ["uploads.github.com"],
      "allowLocalBinding": true
    }
  }
}
```

## Security-reducing options

Several keys explicitly weaken isolation and should be used sparingly: `enableWeakerNestedSandbox` (unprivileged Docker), `enableWeakerNetworkIsolation` (system TLS trust for Go tools behind a MITM proxy), and `allowAppleEvents` (macOS — lets sandboxed commands launch other apps unsandboxed). `allowAppleEvents` is honored only from user, managed, or CLI settings, never project settings.

## Related concepts

- [Settings Files](./settings-files.md) — where the `sandbox` block lives
- [Permission Settings](./permission-settings.md) — `Edit`/`Read`/`WebFetch` rules feed sandbox filesystem and network restrictions
- [Settings Precedence](./settings-precedence.md) — sandbox arrays concatenate and de-duplicate across scopes
