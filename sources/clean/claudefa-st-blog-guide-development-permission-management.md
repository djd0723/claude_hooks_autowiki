---
type: source
source_url: https://claudefa.st/blog/guide/development/permission-management
title: "Claude Code Permissions: Safe vs Fast Development Modes"
raw_capture: ../raw/claudefa-st-blog-guide-development-permission-management.html
captured: 2026-06-29
---

Claude Fast Code Kit 5.5 is here: Fable-optimized, with resumable nested sub-agents and dynamic workflows. Read the playbook.

Claude Fast

Claude Fast

Guide

Learn everything about Claude

Search

⌘K

Who is Claude?What is Claude CodeInstallation GuideNative InstallerFirst ProjectConfiguration BasicsTerminal SetupSandboxing GuideSettings Reference

Mechanics

Development

Large CodebasesAgentic PracticesOpus 4.7 Best PracticesInfraOps VPS GuideGit IntegrationCode ReviewGit WorktreesRemote ControlChannelsScheduled TasksRoutinesPermission ManagementAuto ModeDynamic WorkflowsFable 5 PricingUltracodeWorkflows vs TeamsSubscription Safe UseHigher Usage LimitsFable 5 PricingAgent SDK CreditFeedback LoopsTodo WorkflowsTask ManagementProject TemplatesUsage OptimizationAgent Manager Role

Performance

Agents

SaaS & Startups

Examples & TemplatesTroubleshootingFAQChangelog

SEO Boost

Get Claude Fast

On this page

Development

# Claude Code Permissions: Safe vs Fast Development Modes

Configure Claude Code permissions for your workflow. Learn when to use auto-accept mode and when to maintain strict controls for safety.

Stop configuring. Start shipping.Everything you're reading about and more..
Agentic Orchestration Kit for Claude Code.

Get Claude Fast

Problem: Claude Code asking for permission on every file edit and command kills your flow and burns time.

Quick Win: Press Shift+Tab to cycle through permission modes instantly:

```
# Press Shift+Tab to cycle:
normal → auto-accept edits → plan mode → normal
```

You can also customize this keybinding if Shift+Tab conflicts with your terminal. Now you control your workflow without touching config files. Match your mode to your task and stop the interruption cycle.

## All Permission Modes

Claude Code supports five permission modes, each optimized for different development scenarios. The three core modes cycle via Shift+Tab, while two additional modes are available through configuration.

### Normal Mode (default)

Normal mode prompts for every potentially dangerous operation. You'll see confirmation dialogs for:

- File edits and modifications

- Terminal command execution

- System operations

- Directory changes

This mode prioritizes security over speed, making it perfect for:

- Working on production code

- Unfamiliar codebases

- Learning new techniques

- High-risk operations

### Auto-Accept Mode (acceptEdits)

Auto-accept mode eliminates permission prompts for file edits, enabling uninterrupted execution for the session. Claude proceeds immediately with approved operations.

Activate by pressing Shift+Tab until you see "auto-accept edit on" in the interface.

Best for:

- Large refactoring sessions

- Following well-defined implementation plans

- Research and documentation tasks

- Repetitive operations across multiple files

### Plan Mode (plan)

Plan mode restricts Claude to read-only operations, preventing any modifications while allowing comprehensive analysis.

Perfect for:

- Initial codebase exploration

- Architecture analysis

- Planning complex features

- Code review sessions

### Don't Ask Mode (dontAsk)

Don't Ask mode auto-denies all tool usage unless the tool is explicitly pre-approved via /permissions or your permissions.allow rules. Claude will not prompt you for confirmation. If a tool is not in the allow list, it gets silently denied.

Best for:

- CI/CD pipelines where no human is present to approve

- Locked-down environments with a known set of allowed operations

- Running Claude with a strict, pre-configured permission policy

### Bypass Permissions Mode (bypassPermissions)

Bypass mode skips all permission checks. Claude executes any tool without prompting.

```
# CLI flag equivalent:
claude --dangerously-skip-permissions
```

Only use this in fully isolated environments like containers, VMs, or ephemeral CI runners where Claude cannot cause lasting damage. This mode exists for automation scenarios where the environment itself provides the safety boundary.

Warning: Administrators can disable this mode entirely by setting disableBypassPermissionsMode to "disable" in managed settings. If your organization blocks this mode, neither the setting nor the CLI flag will work.

### Setting a Persistent Default Mode

Instead of cycling modes each session, set your preferred default in settings.json:

```
{
  "defaultMode": "acceptEdits"
}
```

Valid values: default, acceptEdits, plan, dontAsk, bypassPermissions. This saves your preference across sessions so you start in the right mode every time.

## Managing Permissions with /permissions

Instead of manually editing JSON files, use the built-in /permissions command:

```
# Launch the interactive permissions UI
/permissions
```

This interface lets you:

- View currently allowed and denied tools

- Grant permission to specific tools or patterns

- Block access to tools you want to restrict

- See which settings file each rule comes from

- Make changes without restarting Claude Code

## Permission Rule Syntax

Permission rules follow the format Tool or Tool(specifier). Rules live in the permissions object of your settings.json:

```
{
  "permissions": {
    "allow": ["Bash(npm run *)"],
    "deny": ["Bash(rm *)"],
    "ask": ["Bash(git push *)"]
  }
}
```

Three rule types control behavior:

- Allow rules let Claude use the tool without prompting

- Ask rules prompt for confirmation each time

- Deny rules block the tool entirely

Rules evaluate in order: deny first, then ask, then allow. The first matching rule wins, so deny rules always take precedence.

### Matching All Uses of a Tool

Use just the tool name to match all invocations:

RuleEffect

BashMatches all Bash commands

WebFetchMatches all web fetch requests

ReadMatches all file reads

EditMatches all file edits

Bash(*) is equivalent to Bash and matches all Bash commands.

### Bash Wildcard Patterns

Bash rules support glob patterns with *. Wildcards can appear at any position:

```
{
  "permissions": {
    "allow": [
      "Bash(npm run *)",
      "Bash(git commit *)",
      "Bash(git * main)",
      "Bash(* --version)",
      "Bash(* --help *)"
    ],
    "deny": ["Bash(git push *)"]
  }
}
```

Word boundary semantics: The space before * matters. Bash(ls *) matches ls -la but not lsof, while Bash(ls*) matches both.

Shell operator awareness: Claude Code understands shell operators. A rule like Bash(safe-cmd *) will not match safe-cmd && malicious-cmd. This prevents chained command exploitation.

Warning: Bash patterns that constrain command arguments are fragile. For example, Bash(curl http://github.com/ *) intends to restrict curl to GitHub URLs, but won't match variations like options before the URL, different protocols, or variable expansion. For reliable URL filtering, restrict Bash network tools and use WebFetch with domain rules instead.

### Read and Edit Patterns

Read and Edit rules follow gitignore-style path patterns with four distinct types:

PatternMeaningExample

//pathAbsolute path from filesystem rootRead(//Users/alice/secrets/**)

~/pathPath from home directoryRead(~/Documents/*.pdf)

/pathRelative to the settings fileEdit(/src/**/*.ts)

path or ./pathRelative to current directoryRead(*.env)

Important: A pattern like /Users/alice/file is not an absolute path. It resolves relative to your settings file. Use //Users/alice/file for true absolute paths.

In gitignore patterns, * matches files in a single directory while ** matches recursively across directories. To allow all file access, use just the tool name without parentheses: Read, Edit, or Write.

### WebFetch Domain Rules

Control which domains Claude can fetch from:

```
{
  "permissions": {
    "allow": ["WebFetch(domain:docs.anthropic.com)"],
    "deny": ["WebFetch(domain:internal.company.com)"]
  }
}
```

### MCP Tool Patterns

Control MCP server access at the server or tool level:

RuleEffect

mcp__puppeteerMatches any tool from the puppeteer server

mcp__puppeteer__*Same as above (wildcard syntax)

mcp__puppeteer__puppeteer_navigateMatches only the navigate tool specifically

### Task (Subagent) Rules

Control which subagents Claude can spawn using Task(AgentName):

```
{
  "permissions": {
    "deny": ["Task(Explore)"]
  }
}
```

Available agent names include Explore, Plan, and Verify. You can also use the --disallowedTools CLI flag to disable specific agents at startup.

### Extending Permissions with Hooks

PreToolUse hooks run before the permission system and can approve, deny, or modify tool calls at runtime. This gives you programmatic control over permissions beyond static rules. See the Permission Hook guide for a production-ready implementation. ClaudeFast's Code Kit includes a pre-built LLM-powered permission hook that uses a small model to auto-approve safe operations and flag risky ones, so you get uninterrupted flow without blanket bypassPermissions.

## How Permissions Work with Sandboxing

Permissions and sandboxing are complementary security layers providing defense-in-depth:

- Permissions control which tools Claude can use and which files or domains it can access. They apply to all tools (Bash, Read, Edit, WebFetch, MCP, and others).

- Sandboxing provides OS-level enforcement that restricts what Bash commands can access at the filesystem and network level. It applies only to Bash commands and their child processes.

Use both together for the strongest security posture:

- Permission deny rules stop Claude from even attempting to access restricted resources

- Sandbox restrictions prevent Bash commands from reaching resources outside defined boundaries, even if a prompt injection bypasses Claude's decision-making

- Filesystem restrictions in the sandbox use Read and Edit deny rules (not separate sandbox configuration)

- Network restrictions combine WebFetch permission rules with the sandbox's allowedDomains list

Enable sandboxing with the /sandbox command. On macOS it works out of the box using Seatbelt. On Linux and WSL2, install bubblewrap and socat first.

## Managed Permission Settings

For organizations that need centralized control, administrators can deploy managed settings files that cannot be overridden by users or projects:

SettingEffect

allowManagedPermissionRulesOnlyWhen true, only permission rules defined in managed settings apply. User and project rules are ignored.

disableBypassPermissionsModeSet to "disable" to prevent bypassPermissions mode and the --dangerously-skip-permissions CLI flag.

Managed settings file locations:

- macOS: /Library/Application Support/ClaudeCode/managed-settings.json

- Linux/WSL: /etc/claude-code/managed-settings.json

- Windows: C:\Program Files\ClaudeCode\managed-settings.json

These are system-wide paths (not user home directories) requiring administrator privileges. They follow the same format as regular settings files but take the highest precedence in the settings hierarchy.

## Development Scenario Strategies

### Early Development (Use Normal Mode)

When starting new projects or exploring unfamiliar code:

- Keep all permissions manual

- Review each suggested change

- Learn how Claude approaches problems

- Build confidence in the AI's decisions

### Active Development (Use Auto-Accept)

During intensive coding sessions:

- Enable auto-accept for trusted file types

- Allow common commands (npm, git status)

- Maintain prompts for system operations

- Enable uninterrupted workflow

### Code Review (Use Plan Mode)

When analyzing existing codebases:

- Switch to plan mode for safety

- Let Claude explore without modifications

- Generate analysis and recommendations

- Switch modes only when ready to implement

## Common Permission Pitfalls

Over-permissioning: Avoid bypassPermissions mode unless you are running in a fully isolated container or VM. Use dontAsk mode with explicit allow rules for a safer "hands-off" approach.

Under-permissioning: Constantly clicking "Allow" defeats the purpose. Use /permissions to pre-approve repeat operations, or consider the acceptEdits mode for active development sessions.

Mode Confusion: Check your current mode before starting work. The mode indicator appears in the UI. Set defaultMode in settings.json if you always want to start in a specific mode.

Blanket Permissions: Avoid allowing all bash commands. Use specific patterns like Bash(npm run *) to limit scope. Remember that deny rules always win over allow rules.

Fragile Argument Patterns: Do not rely on Bash rules to restrict command arguments (like constraining curl to specific URLs). Use WebFetch domain rules for reliable URL filtering instead.

## What's Next

Master your development workflow by learning complementary techniques:

- Automate permission decisions with hooks and the Permission Hook

- Customize your keybindings for faster mode cycling

- Optimize your feedback loops for faster iteration

- Set up efficient todo workflows for task management

- Configure git integration for seamless version control

- Explore configuration basics for settings.json and advanced setup

Five permission modes for five development scenarios. default for safety, acceptEdits for productivity, plan for exploration, dontAsk for automation, and bypassPermissions for isolated environments. Match the mode to your current needs, and layer sandboxing on top for defense-in-depth.

Last updated on 6/29/2026

Previous

Routines

Next

Auto Mode

### On this page

All Permission Modes

Normal Mode (default)

Auto-Accept Mode (acceptEdits)

Plan Mode (plan)

Don't Ask Mode (dontAsk)

Bypass Permissions Mode (bypassPermissions)

Setting a Persistent Default Mode

Managing Permissions with /permissions

Permission Rule Syntax

Matching All Uses of a Tool

Bash Wildcard Patterns

Read and Edit Patterns

WebFetch Domain Rules

MCP Tool Patterns

Task (Subagent) Rules

Extending Permissions with Hooks

How Permissions Work with Sandboxing

Managed Permission Settings

Development Scenario Strategies

Early Development (Use Normal Mode)

Active Development (Use Auto-Accept)

Code Review (Use Plan Mode)

Common Permission Pitfalls

What's Next

Stop configuring. Start shipping.Everything you're reading about and more..
Agentic Orchestration Kit for Claude Code.

Get Claude Fast

New

Shopify Kit just dropped

Your in-house Shopify x Claude team for Growth, CRO, Paid ads, retention, SEO, ops and Media gen.

Learn more
