# Deep 05 — Session State: What Persists, What Resets, What's Lost

---

## The Fundamental Truth

The Claude API is completely stateless. There is no server-side session.
Claude Code is responsible for ALL state management.

```
  ┌─────────────────────────────────────────────────────────────┐
  │  ANTHROPIC API SERVER                                       │
  │                                                             │
  │  No session storage. No memory between calls.               │
  │  Each API call is treated as entirely independent.          │
  │  The server sees: "Here's a conversation. Reply to it."    │
  └─────────────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────────────┐
  │  CLAUDE CODE (your machine)                                 │
  │                                                             │
  │  Maintains everything:                                      │
  │  • Full message history array                               │
  │  • Tool results from every turn                             │
  │  • File edit history                                        │
  │  • Session transcript on disk                               │
  │  • Memory files                                             │
  └─────────────────────────────────────────────────────────────┘
```

---

## What State Exists in a Session

```
  IN-MEMORY STATE (lost on crash or restart):
  ┌──────────────────────────────────────────────────────────┐
  │  messages[]          the growing conversation array      │
  │  tool_results[]      output from every tool call         │
  │  session_id          unique identifier for this session  │
  │  cwd                 current working directory           │
  │  permission_mode     current permission mode             │
  │  mcp_processes{}     running MCP server processes        │
  │  hook_processes[]    any long-running hooks              │
  └──────────────────────────────────────────────────────────┘

  ON-DISK STATE (survives crash/restart):
  ┌──────────────────────────────────────────────────────────┐
  │  transcript.jsonl    every message/tool call recorded    │
  │  memory/MEMORY.md    persistent memory index             │
  │  memory/*.md         memory topic files                  │
  │  .claude/settings.json  configuration                    │
  │  edited files        changes made by Claude              │
  └──────────────────────────────────────────────────────────┘
```

---

## The Transcript — What Gets Written

Every event in a session is written to:
```
~/.claude/projects/<project-id>/<session-id>/transcript.jsonl
```

JSONL = one JSON object per line. Each line is an event.

The observable events written include:
```
  {"type": "user_message",     "content": "...", "timestamp": "..."}
  {"type": "assistant_message","content": [...], "timestamp": "..."}
  {"type": "tool_use",         "name": "Bash",   "input": {...}}
  {"type": "tool_result",      "output": "...",  "duration_ms": 1234}
  {"type": "session_start",    "session_id": "...", "model": "..."}
  {"type": "session_end",      "timestamp": "..."}
```

This file grows throughout the session. It's the permanent record.

---

## What Happens During /clear

```
  /clear command

          │
          ▼
  messages[] array wiped
          │
          ▼
  New session_id generated
          │
          ▼
  New transcript.jsonl started
          │
          ▼
  CLAUDE.md reloaded (fresh)
          │
          ▼
  MEMORY.md reloaded (fresh)
          │
          ▼
  SessionStart hook fires
          │
          ▼
  Fresh session — Claude has no memory of previous conversation

  ⚠ Memory files (MEMORY.md, topic files) SURVIVE /clear
  ⚠ They are on disk and reload into the new session
```

---

## What Bash State Persists Between Calls

This surprises most users:

```
  Turn 1:
    Bash("cd /tmp && export MY_VAR=hello")
    Result: success

  Turn 2:
    Bash("echo $MY_VAR")
    Result: ""   ← EMPTY! Environment reset.

  Turn 3:
    Bash("pwd")
    Result: /Users/alice/work/myapp  ← NOT /tmp!
```

**Each Bash call runs in a fresh shell process.**
No environment variables, no directory changes, no shell state persists.

```
  What persists between Bash calls:
  ✓ Files written to disk
  ✓ Packages installed (node_modules, etc.)
  ✓ Git commits made
  ✓ Database changes
  ✓ Any filesystem modifications

  What does NOT persist:
  ✗ Environment variables set in previous calls
  ✗ cd (directory changes)
  ✗ Shell variables
  ✗ Shell functions defined
  ✗ Background processes (unless daemonized)
```

**Correct pattern for multi-step commands:**
```bash
# WRONG — second command doesn't see first's env/dir
Bash("export DB_URL=postgres://localhost/mydb")
Bash("node scripts/migrate.js")   ← DB_URL is gone

# RIGHT — chain commands in one call
Bash("export DB_URL=postgres://localhost/mydb && node scripts/migrate.js")

# OR
Bash("cd src && npm run build && npm test")
```

---

## MCP Server Process State

MCP servers are spawned as long-running processes:

```
  Session starts
       │
       ▼
  Claude Code launches MCP server processes:
  GitHub server:  pid 45231  (running)
  Slack server:   pid 45232  (running)
       │
       │  During session...
       │
  Each MCP tool call → sends JSON-RPC to running process
  Process maintains its own state (auth tokens, caches, etc.)
       │
       ▼
  Session ends
  MCP server processes are terminated
  Process state lost (auth tokens, caches)
```

---

## What /compact Does to State

```
  Before /compact:
  messages = [
    turn 1 (user message),
    turn 1 (tool_use: Read),
    turn 1 (tool_result: big file content),   ← will be removed
    turn 2 (tool_use: Edit),
    turn 2 (tool_result: success),
    turn 3 (user message: "also fix the tests"),
    ...50 more turns...
  ]

  After /compact:
  messages = [
    {role: "user", content: "[SUMMARY: Claude read auth.ts, found bug on line 23, edited the expiry from 1h to 24h...]"},
    {role: "assistant", content: "Understood."},
    turn 3 (user message: "also fix the tests"),
    ...recent turns preserved...
  ]

  WHAT'S LOST:
  • The actual file contents that were read (you can re-read them)
  • Detailed intermediate reasoning
  • Tool call specifics from early turns
  • Any instructions given only in early conversation

  WHAT'S PRESERVED:
  • Summary of what happened
  • Recent conversation (last N turns)
  • CLAUDE.md (always reloaded)
  • Memory files (always reloaded)
```

---

## Session Resume

Claude Code sessions can be resumed:

```
  Session A started, work in progress
  User closes terminal

  Later:
  claude --resume <session-id>

       │
       ▼
  transcript.jsonl read from disk
       │
       ▼
  messages[] array reconstructed from transcript
       │
       ▼
  SessionStart hook fires with reason: "resume"
       │
       ▼
  Session continues as if never closed
  (context window state fully restored)
```

---

## The Working Directory and CWD State

```
  Claude Code tracks cwd:
  • Set at session start (where you ran claude)
  • Changes when user cd's in terminal
  • CwdChanged hook fires on change
  • Affects relative path resolution for all tools
  • Does NOT change when Claude runs cd in Bash
    (Bash runs in subprocess, parent cwd unaffected)
```

---

## State Flow Diagram — Full Session

```
  claude starts
       │
       ▼
  ┌─────────────────────────────────────────────────────────────┐
  │  INITIALIZATION                                             │
  │  • session_id = new UUID                                    │
  │  • cwd = current directory                                  │
  │  • messages[] = []                                          │
  │  • load CLAUDE.md → inject into messages                    │
  │  • load MEMORY.md → inject into messages                    │
  │  • start MCP server processes                               │
  │  • SessionStart hook fires                                  │
  └──────────────────────────────┬──────────────────────────────┘
                                 │
                                 ▼
                          User types message
                                 │
                                 ▼
                   UserPromptSubmit hook fires
                                 │
                                 ▼
                   messages.push(user_message)
                                 │
                                 ▼
                          API call (with full messages[])
                                 │
                                 ▼
                   messages.push(assistant_response)
                                 │
                                 ▼
                   Tool calls? → execute → push tool_results
                                 │
                                 ▼
                   Loop until end_turn
                                 │
                                 ▼
                   Stop hook fires
                                 │
                                 ▼
                          Show response to user
                                 │
                                 ▼
                     messages[] grows larger...
                                 │
                                 ▼
                   messages fills ~95% of context
                                 │
                                 ▼
                          /compact triggered
                                 │
                                 ▼
                   PreCompact hook fires
                   Oldest tool results removed
                   Early messages summarized
                   PostCompact hook fires
                                 │
                                 ▼
                     Session continues...
                                 │
                                 ▼
                          User types /clear
                          OR closes terminal
                                 │
                                 ▼
                   SessionEnd hook fires
                   MCP processes terminated
                   Transcript finalized on disk
                   In-memory state lost
```

---

## Summary

```
┌────────────────────────────────────────────────────────────────┐
│  SESSION STATE CHEATSHEET                                      │
├────────────────────────────────────────────────────────────────┤
│  API stateless   : ALL state managed by Claude Code locally    │
│  Bash state      : each call = fresh shell (no env/dir carry)  │
│  Persists        : disk writes, git commits, file edits       │
│  Transcript      : transcript.jsonl written continuously       │
│  /clear          : wipes messages[], starts new session_id    │
│  Memory survives : /clear doesn't delete memory files         │
│  Compact         : removes old tool results, summarizes early  │
│  Resume          : reconstructs messages[] from transcript    │
│  MCP processes   : long-lived per-session, die on session end  │
└────────────────────────────────────────────────────────────────┘
```
