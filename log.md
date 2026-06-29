---
type: log
---

# Build log

_Newest first._

## 2026-06-29 — source task-1: hooks-guide.md (base tier)

**Source:** `sources/clean/code-claude-com-docs-en-hooks-guide-md.md`
**Pipeline stages completed:** clean → foundation → summarize → synthesize → compare → lint → index

**Pages authored:**
- `summaries/hooks-guide.md` — full summary with 7 recipes table, prompt/agent/HTTP hook sections, `if` field, permission mode invariants, debug techniques
- `concepts/hook-lifecycle-events.md` — all 30 hook events and when they fire
- `concepts/hook-types.md` — command, http, mcp_tool, prompt, agent hook types
- `concepts/hook-matchers.md` — matcher field, `if` field, per-event filter table
- `concepts/hook-exit-codes.md` — exit 0/2/other protocol, Stop hook 8-block cap, `stop_hook_active`
- `concepts/hook-scope.md` — configuration location, permission mode interactions
- `synthesis.md` — core thesis: hooks as deterministic enforcement layer; 5 through-lines; 5 open questions
- `comparisons/code-claude-com-docs-en-hooks-guide-md.md` — accuracy matrix; 2 inaccuracies corrected

**Corrections made during compare/lint:** exit code wording (`code other than 0 or 2`, not specifically `1`); event count (30, not 29); frontmatter on index/overview; orphan comparison page linked in index.
