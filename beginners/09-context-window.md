# 09 — Context Window: What Fills It and How Compaction Works

---

## What Is the Context Window?

The context window is Claude's "working memory" for a session.
Everything Claude can currently "see" fits inside it.
Once it's full, either old content is removed or the session can't continue.

```
  ┌─────────────────────────────────────────────────────────────┐
  │                    CONTEXT WINDOW                           │
  │                                                             │
  │  ████████ System prompt                (~2,000 tokens)     │
  │  ████     CLAUDE.md files              (~4,000 tokens)     │
  │  ██       Auto memory (MEMORY.md)      (~2,000 tokens)     │
  │  █        Skill descriptions           (~1,000 tokens)     │
  │  █        MCP tool names               (~500 tokens)       │
  │  ████████████████████ Conversation    (~40,000 tokens)     │
  │  ████████████ Tool results            (~25,000 tokens)     │
  │  ██████ File contents                 (~15,000 tokens)     │
  │  ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░   remaining capacity   │
  └─────────────────────────────────────────────────────────────┘

  Total capacity varies by model:
    Claude Sonnet: ~200,000 tokens
    Claude Opus:   ~200,000 tokens
    Claude Haiku:  ~200,000 tokens
```

---

## Context Composition — What Goes In and When

```
  ┌──────────────────────────────────────────────────────────────┐
  │  1. SYSTEM PROMPT                                            │
  │     Claude Code's hardcoded behavior instructions           │
  │     Loaded: always, first                                    │
  │     Size: fixed (~2,000 tokens)                              │
  └──────────────────────────────────────────────────────────────┘
                │
                ▼
  ┌──────────────────────────────────────────────────────────────┐
  │  2. CLAUDE.md FILES                                          │
  │     All scope levels merged (managed > project > user)       │
  │     Loaded: at session start                                 │
  │     Size: up to 200 lines / 25KB per file                    │
  └──────────────────────────────────────────────────────────────┘
                │
                ▼
  ┌──────────────────────────────────────────────────────────────┐
  │  3. AUTO MEMORY (MEMORY.md)                                  │
  │     Index file only                                          │
  │     Loaded: at session start                                 │
  │     Size: first 200 lines / 25KB                             │
  └──────────────────────────────────────────────────────────────┘
                │
                ▼
  ┌──────────────────────────────────────────────────────────────┐
  │  4. SKILL DESCRIPTIONS                                       │
  │     One-line descriptions of all available skills            │
  │     Loaded: at session start                                 │
  │     Size: ~250 chars × number of skills                      │
  │     Full content: loaded ONLY when skill is invoked          │
  └──────────────────────────────────────────────────────────────┘
                │
                ▼
  ┌──────────────────────────────────────────────────────────────┐
  │  5. MCP TOOL NAMES                                           │
  │     Just the names — NOT definitions                         │
  │     Loaded: at session start                                 │
  │     Definitions: loaded ON DEMAND via tool_search            │
  └──────────────────────────────────────────────────────────────┘
                │
                ▼
  ┌──────────────────────────────────────────────────────────────┐
  │  6. CONVERSATION HISTORY                                     │
  │     All your messages + Claude's responses                   │
  │     Grows: every exchange                                    │
  │     Lost first during compaction: old tool results           │
  └──────────────────────────────────────────────────────────────┘
                │
                ▼
  ┌──────────────────────────────────────────────────────────────┐
  │  7. TOOL RESULTS                                             │
  │     Output from Bash, Read, Grep, MCP calls, etc.           │
  │     Can be large (file contents, command output)             │
  │     First to be cleared during compaction                    │
  └──────────────────────────────────────────────────────────────┘
                │
                ▼
  ┌──────────────────────────────────────────────────────────────┐
  │  8. EXTENDED THINKING (if enabled)                           │
  │     Claude's internal reasoning traces                       │
  │     Only present when thinking mode is on                    │
  └──────────────────────────────────────────────────────────────┘
```

---

## Compaction — When and How It Happens

### Trigger

```
  ┌────────────────────────────────────────────────────────────┐
  │                                                            │
  │  Context fills up...                                       │
  │                                                            │
  │  ████████████████████████████████████░░░  94% full        │
  │  █████████████████████████████████████░   95% ← TRIGGER   │
  │                                                            │
  │  Compaction fires automatically at ~95% capacity           │
  │                                                            │
  │  Override: CLAUDE_AUTOCOMPACT_PCT_OVERRIDE env var         │
  └────────────────────────────────────────────────────────────┘
```

### Compaction Process (Step by Step)

```
  Context at 95%:
  ┌─────────────────────────────────────────┐
  │  System prompt                          │  ← never removed
  │  CLAUDE.md                              │  ← never removed
  │  MEMORY.md                              │  ← never removed
  │  Messages: [user][claude][user][claude] │
  │  Tool result: [huge file content]       │  ← cleared first
  │  Tool result: [bash output]             │  ← cleared first
  │  Tool result: [another file]            │  ← cleared first
  │  Messages: [more conversation]          │
  └─────────────────────────────────────────┘

  Step 1: Clear oldest tool results
  ┌─────────────────────────────────────────┐
  │  System prompt                          │
  │  CLAUDE.md                              │
  │  MEMORY.md                              │
  │  Messages: [user][claude][user][claude] │
  │  [tool result removed]                  │
  │  [tool result removed]                  │
  │  [tool result removed]                  │
  │  Messages: [more conversation]          │
  └─────────────────────────────────────────┘

  Step 2: If still too full → summarize early conversation
  ┌─────────────────────────────────────────┐
  │  System prompt                          │
  │  CLAUDE.md                              │
  │  MEMORY.md                              │
  │  [SUMMARY: earlier conversation...]     │  ← compressed summary
  │  Messages: [recent conversation]        │
  └─────────────────────────────────────────┘
```

### What This Means for You

```
  ⚠ CRITICAL: Instructions given early in a long session
    may disappear after compaction.

  Scenario:
    Start of session:    "Always use snake_case for this project"
    After compaction:    That early instruction is gone
    Result:              Claude reverts to default behavior

  FIX:
    Put important rules in CLAUDE.md (never removed during compaction)
    Don't rely on chat messages for persistent rules
```

---

## Manual Compaction

You can trigger compaction before it becomes critical:

```
  /compact                          ← compact now (no guidance)
  /compact focus on auth module     ← preserve context about auth
  /compact drop test results        ← compact but drop test output
```

---

## Context Visualization

```
  /context
```

Shows a breakdown of what's in your context window:

```
  Context usage: 45,231 / 200,000 tokens (22.6%)

  System prompt:           2,100 tokens
  CLAUDE.md (project):     1,800 tokens
  CLAUDE.md (user):          400 tokens
  Memory (MEMORY.md):        950 tokens
  Skills (12 descriptions):  800 tokens
  MCP tools (names only):    320 tokens

  Conversation:           18,500 tokens
  Tool results:           21,000 tokens

  MCP servers:
    github:               2,500 tokens (tool defs loaded)
    slack:                  800 tokens (2 tools loaded)
```

---

## Deferred Loading — The Efficiency Trick

Both MCP tool definitions and skill full content use deferred loading.

```
  AT SESSION START:
    MCP tools in context:
    "mcp__github__search_repositories" (name only, ~10 tokens)
    "mcp__github__create_pr"          (name only, ~8 tokens)
    "mcp__slack__send_message"        (name only, ~8 tokens)
    ...20 more names...

    Total: ~300 tokens for 20+ MCP tools

  WHEN CLAUDE USES A TOOL:
    Claude calls tool_search("github repositories")
    Full definition loads:
    {
      "name": "mcp__github__search_repositories",
      "description": "Search GitHub repositories...",
      "parameters": {
        "query": { "type": "string", "description": "..." },
        "language": { "type": "string", ... },
        "sort": { ... },
        ...
      }
    }
    Cost: ~500 tokens (one-time, for this tool)
```

---

## What Grows Fastest (Context Killers)

```
  FAST GROWTH:
    Reading many large files      → each file adds thousands of tokens
    Long bash output              → verbose command output fills fast
    Many MCP tool calls           → each result adds to context
    Extended thinking mode        → reasoning traces are very large

  SLOW GROWTH:
    Your chat messages            → usually concise
    Claude's responses            → usually focused
    Skill descriptions            → capped at 250 chars each
```

### Tips to Manage Context

```
  ✓ Use context: fork in skills for file-heavy operations
  ✓ Ask Claude to summarize large outputs before storing
  ✓ Use /compact proactively on long sessions
  ✓ Put persistent rules in CLAUDE.md not in chat
  ✓ Keep MEMORY.md index concise (hits 25KB limit fast)
  ✓ Avoid reading entire large files — ask for specific sections
```

---

## Extended Thinking and Context

When extended thinking is enabled:

```
  Normal response:
    Claude's reasoning: [internal, not stored]
    Response: [stored in context]

  Extended thinking response:
    <thinking>
      [full reasoning trace stored in context]
      can be 10x-50x larger than the response itself
    </thinking>
    Response: [stored in context]
```

This is why extended thinking drains context rapidly on complex tasks.

---

## Summary

```
┌─────────────────────────────────────────────────────────────────┐
│  CONTEXT WINDOW CHEATSHEET                                      │
├─────────────────────────────────────────────────────────────────┤
│  Compaction trigger   : ~95% capacity                           │
│  Override             : CLAUDE_AUTOCOMPACT_PCT_OVERRIDE         │
│  Cleared first        : oldest tool results                     │
│  Then                 : early conversation → summary            │
│  Never removed        : system prompt, CLAUDE.md, memory        │
│  Manual compact       : /compact [focus on <topic>]             │
│  Visualize            : /context                                │
│  MCP loading          : names at start, defs on demand          │
│  Skill loading        : descriptions at start, content on demand│
│  Context killer       : reading large files, verbose bash output│
│  Persistent rules     : ALWAYS put in CLAUDE.md not in chat     │
└─────────────────────────────────────────────────────────────────┘
```
