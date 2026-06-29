---
type: source
source_url: https://claudefa.st/blog/tools/hooks/stop-hook-task-enforcement
title: "Claude Code Stop Hook: Force Task Completion"
raw_capture: ../raw/claudefa-st-blog-tools-hooks-stop-hook-task-enforcement.html
captured: 2026-06-29
---

Claude Fast Code Kit 5.5 is here: Fable-optimized, with resumable nested sub-agents and dynamic workflows. Read the playbook.

Claude Fast

Claude Fast

Tools

Supercharge your Claude

Search

⌘K

Claude Code ToolsKeyboard ShortcutsStatus Line Guide

Hooks

Hooks GuideCross-Platform HooksSetup HooksStop HookSelf-Improving CLAUDE.mdSelf-Validating AgentsSession LifecycleContext Recovery HookSkill Activation HookPermission Hook

Skills

Orchestrators

Monitors

Customization

Resources

MCP & Extensions

Extensions

SEO Boost

Get Claude Fast

On this page

Hooks

# Claude Code Stop Hook: Force Task Completion Before Claude Stops

Use the Stop Hook to ensure Claude finishes tasks before responding. Run tests automatically, validate output, and prevent incomplete work.

Stop configuring. Start shipping.Everything you're reading about and more..
Agentic Orchestration Kit for Claude Code.

Get Claude Fast

Problem: Claude finishes responding but the task is incomplete. Tests are failing. Files are half-written. You ask "are you done?" and Claude says yes, but the build is broken.

Quick Win: Add this Stop hook and Claude cannot stop until tests pass:

```
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "python .claude/hooks/test-gate.py"
          }
        ]
      }
    ]
  }
}
```

```
#!/usr/bin/env python3
import json
import sys
import subprocess

input_data = json.load(sys.stdin)

# CRITICAL: Prevent infinite loops
if input_data.get('stop_hook_active', False):
    sys.exit(0)

# Run tests
result = subprocess.run(['npm', 'test'], capture_output=True, timeout=60)

if result.returncode != 0:
    output = {
        "decision": "block",
        "reason": "Tests are failing. Fix them before completing."
    }
    print(json.dumps(output))
    sys.exit(0)

sys.exit(0)
```

Now Claude literally cannot stop responding until tests pass.

## How the Stop Hook Works

The Stop hook fires every time Claude finishes a response. You can:

- Allow stopping - Exit 0, Claude stops normally

- Block stopping - Return {"decision": "block", "reason": "..."}, Claude continues working

- Run validations - Execute tests, checks, or validations automatically

### The Payload

```
{
  "session_id": "uuid-string",
  "stop_hook_active": false,
  "transcript_path": "/path/to/transcript.jsonl"
}
```

The stop_hook_active flag is critical. When true, Claude is already in a "forced continuation" state from a previous block. Always check this to prevent infinite loops.

## Pattern 1: Test Gate

Block Claude until all tests pass:

```
#!/usr/bin/env python3
import json
import sys
import subprocess

input_data = json.load(sys.stdin)

if input_data.get('stop_hook_active', False):
    sys.exit(0)

result = subprocess.run(
    ['npm', 'test', '--passWithNoTests'],
    capture_output=True,
    timeout=120
)

if result.returncode != 0:
    # Extract last 10 lines of test output for context
    stderr = result.stderr.decode()[-500:] if result.stderr else ""
    print(json.dumps({
        "decision": "block",
        "reason": f"Tests failing. Output: {stderr}"
    }))
    sys.exit(0)

sys.exit(0)
```

## Pattern 2: Build Validation

Ensure the project builds before Claude stops:

```
#!/usr/bin/env python3
import json
import sys
import subprocess

input_data = json.load(sys.stdin)

if input_data.get('stop_hook_active', False):
    sys.exit(0)

result = subprocess.run(
    ['npm', 'run', 'build'],
    capture_output=True,
    timeout=180
)

if result.returncode != 0:
    print(json.dumps({
        "decision": "block",
        "reason": "Build failed. Fix compilation errors before completing."
    }))
    sys.exit(0)

sys.exit(0)
```

## Pattern 3: Lint Check

No stopping with lint errors:

```
#!/usr/bin/env python3
import json
import sys
import subprocess

input_data = json.load(sys.stdin)

if input_data.get('stop_hook_active', False):
    sys.exit(0)

result = subprocess.run(
    ['npx', 'eslint', 'src/', '--max-warnings=0'],
    capture_output=True,
    timeout=60
)

if result.returncode != 0:
    print(json.dumps({
        "decision": "block",
        "reason": "Lint errors detected. Run eslint --fix or resolve manually."
    }))
    sys.exit(0)

sys.exit(0)
```

## Pattern 4: Task Completion Marker

Check if a specific task was completed:

```
#!/usr/bin/env python3
import json
import sys
from pathlib import Path

input_data = json.load(sys.stdin)

if input_data.get('stop_hook_active', False):
    sys.exit(0)

# Check for incomplete task marker
marker = Path('.claude/incomplete-task')
if marker.exists():
    task_info = marker.read_text().strip()
    print(json.dumps({
        "decision": "block",
        "reason": f"Task incomplete: {task_info}. Finish it before stopping."
    }))
    sys.exit(0)

sys.exit(0)
```

Create the marker when starting work:

```
echo "Implement user authentication" > .claude/incomplete-task
```

Delete it when done:

```
rm .claude/incomplete-task
```

## Preventing Infinite Loops

The stop_hook_active flag prevents loops. Here's why it matters:

```
Claude responds → Stop hook fires → "block" → Claude continues
                                            ↓
Claude responds → Stop hook fires → INFINITE LOOP (without flag check)
```

Always check the flag first:

```
if input_data.get('stop_hook_active', False):
    sys.exit(0)  # Allow stopping, break the loop
```

## Combining Multiple Checks

Chain validations in a single hook:

```
#!/usr/bin/env python3
import json
import sys
import subprocess

input_data = json.load(sys.stdin)

if input_data.get('stop_hook_active', False):
    sys.exit(0)

checks = [
    (['npm', 'run', 'lint'], "Lint errors"),
    (['npm', 'run', 'typecheck'], "Type errors"),
    (['npm', 'test'], "Test failures"),
]

for cmd, error_msg in checks:
    result = subprocess.run(cmd, capture_output=True, timeout=120)
    if result.returncode != 0:
        print(json.dumps({
            "decision": "block",
            "reason": f"{error_msg} detected. Fix before completing."
        }))
        sys.exit(0)

sys.exit(0)
```

## When to Use Stop Hooks

Good use cases:

- Ensuring tests pass before "task complete"

- Validating builds succeed

- Checking lint/type errors

- Custom completion criteria

Bad use cases:

- Long-running operations (60 second timeout)

- Network-dependent checks (flaky)

- Blocking on user input (can't interact)

## Configuration

Add to .claude/settings.json:

```
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "python .claude/hooks/stop-validation.py"
          }
        ]
      }
    ]
  }
}
```

Multiple hooks run in parallel. If any returns block, Claude continues.

## Debugging

Infinite loop?

- Check that you're reading stop_hook_active correctly

- Add logging: echo "stop_hook_active: $stop_hook_active" >> ~/.claude/stop-debug.log

Hook not blocking?

- Verify JSON output format: {"decision": "block", "reason": "..."}

- Check exit code is 0 (not 2 for blocking)

Tests timing out?

- 60 second hook timeout

- Run subset of tests or increase efficiency

## The "Ralph Wilgum" Pattern

Named after a community technique, this pattern uses Stop hooks to create persistent task loops:

- Create task marker at session start

- Stop hook blocks until marker removed

- Claude must explicitly complete task to remove marker

- No accidental "I'm done" when work is incomplete

This transforms Claude from "best effort" to "guaranteed completion." The Code Kit takes this further with a BiomeValidator Stop hook that runs lint and format checks on every file Claude touches, combined with a build-then-validate pattern where a dedicated quality-engineer agent inspects the builder's output before marking tasks complete.

## Next Steps

- Set up the main Hooks Guide for all hook types

- Configure Context Recovery to survive compaction

- Learn Skill Activation for automatic skill loading

- Explore Permission Hooks for auto-approval

Last updated on 6/29/2026

Previous

Setup Hooks

Next

Self-Improving CLAUDE.md

### On this page

How the Stop Hook Works

The Payload

Pattern 1: Test Gate

Pattern 2: Build Validation

Pattern 3: Lint Check

Pattern 4: Task Completion Marker

Preventing Infinite Loops

Combining Multiple Checks

When to Use Stop Hooks

Configuration

Debugging

The "Ralph Wilgum" Pattern

Next Steps

Stop configuring. Start shipping.Everything you're reading about and more..
Agentic Orchestration Kit for Claude Code.

Get Claude Fast

New

Shopify Kit just dropped

Your in-house Shopify x Claude team for Growth, CRO, Paid ads, retention, SEO, ops and Media gen.

Learn more
