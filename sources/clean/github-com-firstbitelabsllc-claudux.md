---
type: source
source_url: https://github.com/firstbitelabsllc/claudux
title: "GitHub - firstbitelabsllc/claudux: AI-powered documentation generator for your codebase using Claude Code and VitePress · GitHub"
raw_capture: ../raw/github-com-firstbitelabsllc-claudux.html
captured: 2026-06-29
---

Skip to content

## Navigation Menu

Toggle navigation

Sign in

Appearance settings

-

Platform

-

AI CODE CREATION

-

GitHub CopilotWrite better code with AI

-

GitHub Copilot appDirect agents from issue to merge

-

MCP RegistryNewIntegrate external tools

-

DEVELOPER WORKFLOWS

-

ActionsAutomate any workflow

-

CodespacesInstant dev environments

-

IssuesPlan and track work

-

Code ReviewManage code changes

-

APPLICATION SECURITY

-

GitHub Advanced SecurityFind and fix vulnerabilities

-

Code securitySecure your code as you build

-

Secret protectionStop leaks before they start

-

EXPLORE

- Why GitHub

- Documentation

- Blog

- Changelog

- Marketplace

View all features

-

Solutions

-

BY COMPANY SIZE

- Enterprises

- Small and medium teams

- Startups

- Nonprofits

-

BY USE CASE

- App Modernization

- DevSecOps

- DevOps

- CI/CD

- View all use cases

-

BY INDUSTRY

- Healthcare

- Financial services

- Manufacturing

- Government

- View all industries

View all solutions

-

Resources

-

EXPLORE BY TOPIC

- AI

- Software Development

- DevOps

- Security

- View all topics

-

EXPLORE BY TYPE

- Customer stories

- Events & webinars

- Ebooks & reports

- Business insights

- GitHub Skills

-

SUPPORT & SERVICES

- Documentation

- Customer support

- Community forum

- Trust center

- Partners

View all resources

-

Open Source

-

COMMUNITY

-

GitHub SponsorsFund open source developers

-

PROGRAMS

- Security Lab

- Maintainer Community

- Accelerator

- GitHub Stars

- Archive Program

-

REPOSITORIES

- Topics

- Trending

- Collections

-

Enterprise

-

ENTERPRISE SOLUTIONS

-

Enterprise platformAI-powered developer platform

-

AVAILABLE ADD-ONS

-

GitHub Advanced SecurityEnterprise-grade security features

-

Copilot for BusinessEnterprise-grade AI features

-

Premium SupportEnterprise-grade 24/7 support

- Pricing

Search or jump to...

# Search code, repositories, users, issues, pull requests...

-->

Search

Clear

Search syntax tips

#
Provide feedback

-->

We read every piece of feedback, and take your input very seriously.

Include my email address so I can be contacted

Cancel

Submit feedback

#
Saved searches

## Use saved searches to filter your results more quickly

-->

Name

Query

To see all available qualifiers, see our documentation.

Cancel

Create saved search

Sign in

Sign up

Appearance settings

Resetting focus

You signed in with another tab or window. Reload to refresh your session.
You signed out in another tab or window. Reload to refresh your session.
You switched accounts on another tab or window. Reload to refresh your session.

Dismiss alert

{{ message }}

### Uh oh!

There was an error while loading. Please reload this page.

firstbitelabsllc

/

claudux

Public

-
Notifications
You must be signed in to change notification settings

-
Fork
0

-

Star
4

-

Code

-

Issues
0

-

Pull requests
1

-

Actions

-

Security and quality
0

-

Insights

Additional navigation options

-

Code

-

Issues

-

Pull requests

-

Actions

-

Security and quality

-

Insights

# firstbitelabsllc/claudux

main

2 Branches0 Tags

Go to file

Code

Open more actions menu

## Folders and files

NameName

Last commit message

Last commit date

## Latest commit

leojkwan

fix: harden claudux section patch regen

success

May 23, 2026

3f5fcec · May 23, 2026

## History

3 Commits

Open commit details

3 Commits

.github

.github

Initial claudux release

May 23, 2026

assets

assets

Initial claudux release

May 23, 2026

bin

bin

Initial claudux release

May 23, 2026

docs

docs

fix: harden claudux section patch regen

May 23, 2026

lib

lib

fix: harden claudux section patch regen

May 23, 2026

scripts

scripts

Initial claudux release

May 23, 2026

tests

tests

fix: harden claudux section patch regen

May 23, 2026

.gitignore

.gitignore

Initial claudux release

May 23, 2026

.npmignore

.npmignore

Initial claudux release

May 23, 2026

ARCHITECTURE.md

ARCHITECTURE.md

Initial claudux release

May 23, 2026

CHANGELOG.md

CHANGELOG.md

Initial claudux release

May 23, 2026

CONTRIBUTING.md

CONTRIBUTING.md

Initial claudux release

May 23, 2026

LICENSE

LICENSE

Initial claudux release

May 23, 2026

README.md

README.md

fix: harden claudux section patch regen

May 23, 2026

SECURITY.md

SECURITY.md

Initial claudux release

May 23, 2026

claude.md

claude.md

Initial claudux release

May 23, 2026

claudux.json

claudux.json

Initial claudux release

May 23, 2026

claudux.md

claudux.md

Initial claudux release

May 23, 2026

docs-structure.json

docs-structure.json

Initial claudux release

May 23, 2026

package-lock.json

package-lock.json

Initial claudux release

May 23, 2026

package.json

package.json

Initial claudux release

May 23, 2026

View all files

## Repository files navigation

- README

- Contributing

- MIT license

- Security

More items

# claudux

Claudux is a local CLI for generating and maintaining VitePress documentation from a codebase. It can use Claude by default or Codex with CLAUDUX_BACKEND=codex.

Current versions also support deterministic docs mode: when a repository checks in docs-structure.json, claudux treats the docs tree as source-owned state, builds a static analysis index before model invocation, and applies model output through bounded section patches instead of broad rewrites.

See it in action: Live documentation (generated by claudux itself)

## Install

```
npm install -g claudux
```

Requirements:

- Node.js >= 18

- An authenticated AI CLI:

- Claude CLI for the default backend

- Codex CLI for CLAUDUX_BACKEND=codex

Claudux runs as a local shell tool. The selected AI CLI still receives the prompt context claudux gives it, according to that backend's own authentication, transport, and data handling behavior.

## Quick Start

```
cd your-project

# Generate or update docs.
claudux update

# Preview the VitePress site.
claudux serve

# Validate generated docs without regenerating.
claudux validate
```

Use a focused directive when you know what changed:

```
claudux update -m "Document the new authentication flow"
claudux update --with "Refresh CLI command docs from bin/claudux"
```

## How It Works

Claudux is command-first: most commands inspect or serve local docs, and only update, template, or the interactive menu need an AI backend.

Loading

```
flowchart TD
    CLI["claudux"] --> COMMAND{"command"}
    COMMAND --> MENU["interactive menu"]
    COMMAND --> UPDATE["update [-m|--with] [--strict]"]
    COMMAND --> AUDIT["audit [--json|--strict|--release|--handoff-strict]"]
    COMMAND --> VALIDATE["validate"]
    COMMAND --> SERVE["serve / status / diff / check"]

    UPDATE --> SCAN["cleanup + detect project + build index"]
    SCAN --> MANIFEST{"docs-structure.json?"}
    MANIFEST -->|no| CLASSIC["classic docs update"]
    MANIFEST -->|yes| INDEX["section patch contract + impacted sections"]
    INDEX --> PATCH["backend returns section patch JSON"]
    PATCH --> APPLY["validate and apply bounded patches"]
    CLASSIC --> VERIFY["post-generation validation"]
    APPLY --> VERIFY
    VERIFY --> STATE["checkpoint"]

```

Deterministic mode is meant for larger repos where the structure of the documentation is part of the product:

- docs-structure.json owns page IDs, paths, navigation order, source ownership, deletion policy, and required sections.

- .claudux/index/static-analysis.json records sorted, hash-based facts about tracked source files, docs files, package scripts, shell dependencies, headings, links, and manifest ownership.

- .claudux/index/impacted-docs.json maps changed files through manifest source_patterns and dependency edges to the docs that may be stale.

- The model returns JSON between CLAUDUX_SECTION_PATCHES_JSON_START and CLAUDUX_SECTION_PATCHES_JSON_END.

- Claudux applies only validated patches to manifest-owned generated sections.

- Pinned sections, read-only sections, and skip-marker blocks are guarded before and after generation.

- AI cleanup paths and claudux recreate are blocked from deleting manifest-owned docs unless explicitly unlocked with environment overrides.

The model can propose wording. The repository owns structure.

For team-agent handoffs and CI checks, claudux audit provides a no-AI readiness snapshot. It reports project detection, manifest validity, link status, checkpoint freshness, changed files since checkpoint, and uncommitted docs/config changes; --json makes the same report machine-readable.

## Backends

Backend
Select with
CLI required
Notes

Claude
default or CLAUDUX_BACKEND=claude
claude
Default backend. Uses FORCE_MODEL=sonnet unless overridden.

Codex
CLAUDUX_BACKEND=codex
codex
Supported backend. Defaults to gpt-5.4 with xhigh reasoning.

```
# Default: Claude
claudux update

# Codex
CLAUDUX_BACKEND=codex claudux update

# Codex settings
export CLAUDUX_BACKEND=codex
export CODEX_MODEL=gpt-5.4
export CODEX_REASONING_EFFORT=xhigh
claudux update
```

Codex runs through codex exec --json with approval_policy="never". Its default sandbox is danger-full-access; when claudux enters manifest section-patch mode, it requests a read-only sandbox unless CODEX_SANDBOX_MODE is set.

## Commands

```
claudux                 # Interactive menu
claudux update          # Generate or update docs
claudux update -m "..." # Update with a focused directive
claudux update --strict # Fail if internal links remain broken
claudux serve           # Start the VitePress dev server
claudux diff            # Show files changed since the last checkpoint
claudux status          # Show documentation freshness
claudux audit           # Print a no-AI readiness report for humans, CI, and agents
claudux audit --release # Fail docs/package release-readiness drift
claudux validate        # Validate manifest and internal links
claudux check           # Verify Node, backend CLI, and docs state
claudux template        # Generate claudux.md preferences
claudux recreate        # Delete docs and regenerate, unless manifest guard blocks it
claudux --version       # Show installed version
claudux --help          # Show help
```

claudux diff and claudux status read .claudux-state.json, which is local per developer and ignored by git.

## Configuration

Optional claudux.json in the project root:

```
{
  "project": {
    "name": "Your Project",
    "type": "react"
  }
}
```

Common config files:

- claudux.json sets project metadata and type overrides.

- claudux.md stores documentation preferences generated by claudux template.

- docs-structure.json is the deterministic manifest for repos that want pinned pages, source-owned sections, bounded patching, and deletion guards.

Claudux auto-detects common project types from repo files, including React, Next.js, JavaScript, Node.js, Python, Go, iOS, Android, Rust, Rails, and Flutter.

## Content Protection

Claudux automatically protects sensitive paths such as notes/, private/, .git/, node_modules/, *.env, *.key, and *.pem.

Use skip markers to protect specific blocks:

```
<!-- skip -->
This block is preserved by claudux.
<!-- /skip -->
```

Language-specific marker pairs are supported for common source files, including // skip, # skip, /* skip */, and -- skip.

In deterministic mode, skip-marker hashes are captured in the guard snapshot so protected blocks cannot be changed silently during generation.

## How Claudux Compares

Claudux
TypeDoc / JSDoc
Docusaurus
Manual docs

Input
Source code, existing docs, optional manifest
Type annotations and comments
Hand-written docs
Hand-written docs

Output
VitePress site
API reference
Static docs site
Any format

Maintenance
Re-run claudux update
Rebuild on code change
Edit by hand
Edit by hand

AI backend
Claude or Codex CLI
None
None by default
None

Link validation
Built in
Generator-specific
Build-time
Manual

Structure guardrails
Optional manifest, pinned sections, bounded patches
No
Config-owned navigation
Manual review

Claudux is not an API reference generator. It is designed for guides, architecture pages, command references, and source-aware maintenance docs. It can run alongside TypeDoc, JSDoc, or other reference generators.

## Demo

## Project Docs

- Live docs

- Architecture

- Deterministic generation

## Sibling Project

vidux is First Bite Labs' private plan-first orchestration system for AI coding agents. Claudux handles docs; vidux handles planning. When claudux detects vidux.config.json or a root PLAN.md with projects/*/PLAN.md, it uses the vidux template so team-agent plans, drift logs, command surfaces, and verification gates become first-class documentation inputs.

## License

MIT

## About

AI-powered documentation generator for your codebase using Claude Code and VitePress

firstbitelabsllc.github.io/claudux/

### Topics

cli

documentation

docs

ai

static-site

developer-tools

cleanup

claude

vitepress

doc-generator

link-check

anthropic

claude-code

### Resources

Readme

### License

MIT license

### Contributing

Contributing

### Security policy

Security policy

### Uh oh!

There was an error while loading. Please reload this page.

Activity

Custom properties

### Stars

4
stars

### Watchers

0
watching

### Forks

0
forks

Report repository

##
Releases

No releases published

##
Packages
0

No packages published

### Uh oh!

There was an error while loading. Please reload this page.

##
Contributors
3

-

leojkwan
Leo Kwan

-

claude
Claude

-

lkwan-sc

## Languages

-

Shell
95.0%

-

Vue
1.8%

-

TypeScript
1.6%

-

CSS
1.4%

-

JavaScript
0.2%

## Footer

© 2026 GitHub, Inc.

### Footer navigation

-
Terms

-
Privacy

-
Security

-
Status

-
Community

-
Docs

-
Contact

-

Manage cookies

-

Do not share my personal information

You can’t perform that action at this time.
