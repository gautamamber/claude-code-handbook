# Deep 03 — Subagents: How Agent Spawning Works Internally

---

## What Is a Subagent?

A subagent is a completely separate Claude API session spawned by the parent session.
It has its own context window, its own tool calls, and its own agentic loop.

```
  ┌────────────────────────────────────────────────────────────┐
  │  PARENT SESSION                                            │
  │  model: claude-opus-4-6                                    │
  │  context: 80,000 tokens                                    │
  │                                                            │
  │  User: "Refactor the entire auth module"                   │
  │  Claude: I'll spawn specialized subagents...               │
  │                                                            │
  │  ┌──────────────────────────────┐                         │
  │  │  SUBAGENT: Explore           │  ← fresh context         │
  │  │  Task: analyze auth code     │    fresh tool calls      │
  │  │  model: (configured)         │    own agentic loop      │
  │  │  Returns: analysis summary   │                          │
  │  └──────────────────────────────┘                         │
  │                                                            │
  │  Parent receives: summary of analysis                      │
  │  (Not the full subagent context — just the result)         │
  └────────────────────────────────────────────────────────────┘
```

---

## The Agent Tool — How It's Called

```json
{
  "type": "tool_use",
  "id": "toolu_01AgentXyz",
  "name": "Agent",
  "input": {
    "description": "Analyze auth module",
    "prompt": "Analyze the auth module in src/auth/. Identify:\n1. All exported functions\n2. Dependencies used\n3. Security vulnerabilities\n4. Test coverage gaps\nReturn a structured report.",
    "subagent_type": "Explore",
    "model": "sonnet"
  }
}
```

---

## Parent → Subagent Context Flow

What context does a subagent get? This is nuanced:

```
  PARENT SESSION HAS:
  • Full conversation history (80,000 tokens)
  • All tool results from prior calls
  • CLAUDE.md content
  • Memory from MEMORY.md

  SUBAGENT GETS:
  • Its own fresh system prompt
  • CLAUDE.md content (reloaded)
  • Memory (reloaded)
  • The prompt passed in the Agent tool call
  • Nothing else from parent's conversation history

  ┌──────────────────────────────────────────────────────────┐
  │  This is deliberate — subagents are ISOLATED.            │
  │                                                          │
  │  Parent cannot leak sensitive info to subagent           │
  │  via context. Only the explicit prompt is passed.        │
  │                                                          │
  │  Subagent cannot read parent's file edits in progress,   │
  │  earlier messages, or intermediate tool results.         │
  └──────────────────────────────────────────────────────────┘
```

---

## Subagent Agentic Loop

The subagent runs its OWN complete agentic loop:

```
  Parent calls Agent tool
          │
          ▼
  Subagent spawned (fresh session)
          │
          ▼
  ┌──────────────────────────────────────────────┐
  │  SUBAGENT AGENTIC LOOP                       │
  │                                              │
  │  Turn 1: Analyze prompt → tool_use (Glob)    │
  │  Turn 2: tool_result → tool_use (Read x3)   │
  │  Turn 3: tool_result → tool_use (Grep)       │
  │  Turn 4: tool_result → end_turn              │
  │                                              │
  │  Subagent builds full report                 │
  └──────────────────────────┬───────────────────┘
                             │
                             ▼
  Subagent's FINAL RESPONSE text injected as tool_result
  into parent's tool_result block
          │
          ▼
  Parent sees: "Auth module analysis: [summary]..."
  Parent continues its own agentic loop
```

---

## What the Parent Sees Back

The parent receives ONLY the final text response from the subagent.
Not the tool calls. Not intermediate steps. Just the answer.

```json
{
  "role": "user",
  "content": [
    {
      "type": "tool_result",
      "tool_use_id": "toolu_01AgentXyz",
      "content": "Auth Module Analysis:\n\n**Exported Functions:**\n- generateToken(userId): string\n- verifyToken(token): Payload | null\n- refreshToken(token): string\n\n**Dependencies:** jsonwebtoken, bcrypt\n\n**Security Issues Found:**\n1. JWT secret loaded from env without validation (token.ts:5)\n2. No rate limiting on token generation\n\n**Test Coverage:** 67% (missing refresh token edge cases)"
    }
  ]
}
```

---

## Subagent Cannot Spawn Subagents

This is a hard architectural limit:

```
  Parent session
      │
      ├── spawns Subagent A  ✓
      ├── spawns Subagent B  ✓
      └── spawns Subagent C  ✓

  Subagent A
      ├── tries to spawn Subagent A1  ✗  BLOCKED
      └── (Agent tool calls return error)

  Why? Prevents runaway recursive agent chains.
  Nesting depth is limited to 1 level.
```

---

## Background Subagents

Subagents can run in the background while the parent continues:

```json
{
  "name": "Agent",
  "input": {
    "description": "Run test suite",
    "prompt": "Run npm test and report results",
    "run_in_background": true
  }
}
```

```
  Parent calls Agent (background)
          │
          ▼
  Parent immediately receives:
  tool_result: { "agent_id": "agent-abc123", "status": "started" }
          │
          ▼
  Parent continues its own work
  (doesn't wait for subagent)
          │
          │
  ... later ...
          │
          ▼
  TaskCompleted hook fires
  Parent receives notification: subagent finished
```

---

## Resuming Subagents

A subagent can be resumed with its agent_id:

```json
{
  "name": "Agent",
  "input": {
    "description": "Continue test run",
    "prompt": "What were the failures from the previous run?",
    "resume": "agent-abc123"
  }
}
```

The resumed subagent has access to its previous conversation transcript.
It continues from where it left off.

---

## Subagent Types and Their Capabilities

```
  ┌────────────────────────────────────────────────────────────┐
  │  general-purpose                                           │
  │  • All tools available                                     │
  │  • Best for: complex multi-step tasks                      │
  │  • Default if subagent_type not specified                  │
  └────────────────────────────────────────────────────────────┘

  ┌────────────────────────────────────────────────────────────┐
  │  Explore                                                   │
  │  • Read-only tools: Read, Glob, Grep, WebFetch, WebSearch  │
  │  • NO Edit, Write, Bash, Agent                             │
  │  • Best for: codebase research, finding files, analysis    │
  │  • Fast — no permission prompts for writes                 │
  └────────────────────────────────────────────────────────────┘

  ┌────────────────────────────────────────────────────────────┐
  │  Plan                                                      │
  │  • All read tools + plan creation                          │
  │  • Produces structured implementation plan                 │
  │  • Best for: "how should we implement X?" questions        │
  └────────────────────────────────────────────────────────────┘
```

---

## Permission Inheritance

```
  Parent mode: acceptEdits
      │
      ▼
  Subagent inherits: acceptEdits
      │
      ├── Can override via permissionMode in agent frontmatter
      │   (unless parent is bypassPermissions)
      │
      └── Classifier (if auto mode) runs separately for subagent

  Parent mode: bypassPermissions
      │
      ▼
  Subagent inherits: bypassPermissions
      └── Cannot be overridden to anything stricter
```

---

## Worktree Isolation

When `isolation: "worktree"` is set, the subagent gets its own git branch:

```
  Parent: working in /Users/alice/work/myapp/ (branch: main)
          │
          │ Agent tool called with isolation: "worktree"
          ▼
  WorktreeCreate hook fires
          │
          ▼
  Subagent working in:
    /tmp/worktrees/myapp-a1b2c3/ (branch: claude/fix-auth-bug)
          │
  Subagent edits files in its isolated directory
  Cannot affect parent's working files
          │
          ▼
  Subagent ends
          │
      ┌───┴───┐
   no changes  changes made
      │              │
      ▼              ▼
  Auto-cleanup  Branch preserved:
                claude/fix-auth-bug
                Path returned to parent
```

---

## Transcript Storage

Each subagent's conversation is stored separately:

```
  ~/.claude/projects/<project-id>/
  └── <parent-session-id>/
      ├── transcript.jsonl               ← parent conversation
      └── subagents/
          ├── agent-abc123.jsonl         ← subagent 1 transcript
          └── agent-def456.jsonl         ← subagent 2 transcript
```

If the parent session undergoes compaction, the subagent transcripts are unaffected.
They persist independently and can be examined after the session.

---

## Skills with context: fork

When a skill has `context: fork`, it internally uses the Agent tool:

```
  User: /security-review

  Without context: fork:
    Skill content loaded into parent context
    Tool calls run in parent session
    All results pollute parent context window

  With context: fork:
    Agent tool called internally with skill prompt
    Skill runs in isolated subagent
    Only final summary returned to parent
    Parent context window stays clean
```

---

## SubagentStart / SubagentStop Hooks

```json
// .claude/settings.json
{
  "hooks": {
    "SubagentStart": [
      {
        "matcher": "",
        "hooks": [{
          "type": "command",
          "command": "echo 'Subagent started' >> /tmp/agent-log.txt",
          "async": true
        }]
      }
    ],
    "SubagentStop": [
      {
        "matcher": "",
        "hooks": [{
          "type": "command",
          "command": "echo 'Subagent stopped' >> /tmp/agent-log.txt",
          "async": true
        }]
      }
    ]
  }
}
```

Hook payload for SubagentStart includes:
- `agent_id` — unique ID for this subagent
- `agent_type` — Explore / Plan / general-purpose
- `session_id` — parent session ID

---

## Why Subagents Exist — The Context Window Problem

```
  Without subagents:
    Parent reads 50 files → fills context window
    Parent can't continue complex multi-file tasks
    Context compaction loses early file contents

  With subagents:
    Parent spawns Explore subagent: "read all auth files, summarize"
    Subagent uses ITS context window for all the file reading
    Parent receives compact summary (1,000 tokens)
    Parent's context window barely affected

  Subagents are context window amplifiers.
```

---

## Summary

```
┌────────────────────────────────────────────────────────────────┐
│  SUBAGENTS CHEATSHEET                                          │
├────────────────────────────────────────────────────────────────┤
│  Context       : fresh — doesn't see parent's history          │
│  Returns       : only final response text (not tool calls)     │
│  Nesting       : max 1 level (subagents can't spawn subagents) │
│  Background    : run_in_background: true                       │
│  Resume        : resume: "agent-id" to continue               │
│  Worktree      : isolation: "worktree" for git isolation       │
│  Transcripts   : stored separately, survive compaction         │
│  Hooks         : SubagentStart / SubagentStop                  │
│  Permissions   : inherited from parent, can override           │
│  Context value : amplifies effective context via delegation    │
└────────────────────────────────────────────────────────────────┘
```
