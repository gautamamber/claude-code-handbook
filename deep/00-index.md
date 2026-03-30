# Deep Internals — Index

> This folder covers HOW Claude Code works internally.
> Not the features — the mechanics underneath.

---

## Files in This Folder

| File | What It Covers |
|------|---------------|
| `01-agentic-loop.md` | The core loop: API calls, tool_use/tool_result cycle, stop_reason, streaming |
| `02-tool-protocol.md` | Tool definitions, call format, result injection, parallel calls, error handling |
| `03-subagents.md` | Subagent spawning, context isolation, worktree isolation, background agents |
| `04-extended-thinking.md` | Thinking blocks, budget_tokens, context cost, encrypted thinking |
| `05-session-state.md` | What persists vs resets, Bash state, transcript, /compact, session resume |

---

## The One Diagram That Explains Everything

```
  USER MESSAGE
       │
       ▼
  ┌──────────────────────────────────────────────────────┐
  │  Build messages[] array:                             │
  │  [system prompt + CLAUDE.md + memory + history]      │
  └──────────────────────────────────────────────────────┘
       │
       ▼
  POST /v1/messages  (full history every call)
       │
       ▼
  Response stream...
       │
       ├── stop_reason: end_turn ──► show to user, done
       │
       └── stop_reason: tool_use
                │
                ▼
           Extract tool_use block
                │
                ▼
           PreToolUse hook → permission check
                │
                ▼
           Execute tool (Bash/Read/Edit/MCP/Agent...)
                │
                ▼
           PostToolUse hook
                │
                ▼
           Append tool_result to messages[]
                │
                └──────────────────────────── loop back to POST
```

---

## Key Mental Models

**1. The API is stateless.**
Every call sends the FULL conversation. Claude Code owns state management.

**2. Tool results are user messages.**
`tool_result` blocks go in `role: "user"` messages. The turn structure is:
`user → assistant (tool_use) → user (tool_result) → assistant → ...`

**3. Bash runs fresh every time.**
No `cd`, `export`, or shell state persists between Bash calls. Chain commands with `&&`.

**4. Subagents are isolated.**
They get your prompt but not your conversation history. Only final text returns.

**5. Context window = API cost.**
Every token in messages[] is re-sent every call. Large files = expensive loops.

**6. Hooks are the deterministic layer.**
Unlike Claude's probabilistic decisions, hooks exit 2 = hard block, no exceptions.

---

## What's NOT Documented (Proprietary)

```
  • Exact JSONL transcript schema
  • Precise context building algorithm (injection order)
  • Internal iteration limits per session
  • How subagent communication wire protocol works
  • The exact auto-mode classifier prompt
  • Tool_search deferred loading API mechanism
  • Retry/backoff exact parameters
```

These are implementation details Anthropic keeps internal.
The files above cover everything that IS documented or definitively observable.
