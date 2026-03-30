# Deep 02 — Tool Protocol: How Tools Actually Work

---

## Overview

Every tool in Claude Code — Bash, Read, Edit, Write, Grep, Glob, Agent, MCP tools —
uses the same Anthropic tool_use protocol. Understanding this protocol reveals
exactly how Claude interacts with your machine.

---

## The Complete Tool Call Lifecycle

```
  ┌─────────────────────────────────────────────────────────────┐
  │  1. TOOL DEFINITION                                         │
  │     Sent in every API request                               │
  │     Tells Claude what tools exist and how to call them      │
  └──────────────────────────────┬──────────────────────────────┘
                                 │
                                 ▼
  ┌─────────────────────────────────────────────────────────────┐
  │  2. TOOL SELECTION                                          │
  │     Claude decides to use a tool                            │
  │     Emits tool_use content block in response                │
  └──────────────────────────────┬──────────────────────────────┘
                                 │
                                 ▼
  ┌─────────────────────────────────────────────────────────────┐
  │  3. HOOK GATE (PreToolUse)                                  │
  │     Claude Code intercepts before execution                 │
  │     Hook scripts run → can block with exit 2               │
  └──────────────────────────────┬──────────────────────────────┘
                                 │
                                 ▼
  ┌─────────────────────────────────────────────────────────────┐
  │  4. PERMISSION CHECK                                        │
  │     allow/deny/ask rules evaluated                          │
  │     Mode-specific behavior applied                          │
  └──────────────────────────────┬──────────────────────────────┘
                                 │
                                 ▼
  ┌─────────────────────────────────────────────────────────────┐
  │  5. EXECUTION                                               │
  │     Tool runs in Claude Code process                        │
  │     Output captured                                         │
  └──────────────────────────────┬──────────────────────────────┘
                                 │
                                 ▼
  ┌─────────────────────────────────────────────────────────────┐
  │  6. HOOK GATE (PostToolUse / PostToolUseFailure)            │
  │     After execution hooks run                               │
  │     Can trigger linting, logging, notifications            │
  └──────────────────────────────┬──────────────────────────────┘
                                 │
                                 ▼
  ┌─────────────────────────────────────────────────────────────┐
  │  7. RESULT INJECTION                                        │
  │     tool_result block appended to messages array           │
  │     Claude sees output in next API call                     │
  └──────────────────────────────┬──────────────────────────────┘
                                 │
                                 ▼
                         Next API call (loop)
```

---

## Tool Definition Format

Sent in every API request's `tools` array:

```json
{
  "name": "Bash",
  "description": "Execute a bash command in the terminal. Use for running tests, installing packages, executing scripts, and other shell operations.",
  "input_schema": {
    "type": "object",
    "properties": {
      "command": {
        "type": "string",
        "description": "The bash command to execute. Must be a valid shell command."
      },
      "timeout": {
        "type": "integer",
        "description": "Maximum time to wait in milliseconds. Default: 120000 (2 minutes)"
      },
      "description": {
        "type": "string",
        "description": "A short description of what this command does, shown to user"
      }
    },
    "required": ["command"]
  }
}
```

---

## All Built-in Tool Schemas (Inferred from Behavior)

### Read

```json
{
  "name": "Read",
  "input_schema": {
    "properties": {
      "file_path": { "type": "string" },
      "offset":    { "type": "integer", "description": "Line to start from" },
      "limit":     { "type": "integer", "description": "Number of lines to read" }
    },
    "required": ["file_path"]
  }
}
```

### Edit

```json
{
  "name": "Edit",
  "input_schema": {
    "properties": {
      "file_path":   { "type": "string" },
      "old_string":  { "type": "string", "description": "Exact text to replace" },
      "new_string":  { "type": "string", "description": "Replacement text" },
      "replace_all": { "type": "boolean", "description": "Replace all occurrences" }
    },
    "required": ["file_path", "old_string", "new_string"]
  }
}
```

### Write

```json
{
  "name": "Write",
  "input_schema": {
    "properties": {
      "file_path": { "type": "string" },
      "content":   { "type": "string" }
    },
    "required": ["file_path", "content"]
  }
}
```

### Glob

```json
{
  "name": "Glob",
  "input_schema": {
    "properties": {
      "pattern": { "type": "string", "description": "Glob pattern e.g. **/*.ts" },
      "path":    { "type": "string", "description": "Directory to search in" }
    },
    "required": ["pattern"]
  }
}
```

### Grep

```json
{
  "name": "Grep",
  "input_schema": {
    "properties": {
      "pattern":     { "type": "string" },
      "path":        { "type": "string" },
      "glob":        { "type": "string" },
      "type":        { "type": "string", "description": "File type: js, py, ts..." },
      "output_mode": { "type": "string", "enum": ["content", "files_with_matches", "count"] },
      "-i":          { "type": "boolean", "description": "Case insensitive" },
      "-n":          { "type": "boolean", "description": "Show line numbers" },
      "-A":          { "type": "integer", "description": "Lines after match" },
      "-B":          { "type": "integer", "description": "Lines before match" },
      "-C":          { "type": "integer", "description": "Context lines" },
      "head_limit":  { "type": "integer" },
      "multiline":   { "type": "boolean" }
    },
    "required": ["pattern"]
  }
}
```

### Agent

```json
{
  "name": "Agent",
  "input_schema": {
    "properties": {
      "description":   { "type": "string", "description": "3-5 word description" },
      "prompt":        { "type": "string", "description": "Full task for the subagent" },
      "subagent_type": { "type": "string", "enum": ["general-purpose", "Explore", "Plan", ...] },
      "isolation":     { "type": "string", "enum": ["worktree"] },
      "model":         { "type": "string" },
      "resume":        { "type": "string", "description": "Agent ID to resume" },
      "run_in_background": { "type": "boolean" }
    },
    "required": ["description", "prompt"]
  }
}
```

---

## A Real Bash Tool Call — Full Trace

```
  Claude wants to run: npm test

  ─── Claude's response (API output) ──────────────────────────

  {
    "role": "assistant",
    "content": [
      {
        "type": "text",
        "text": "Let me run the test suite to see the current state."
      },
      {
        "type": "tool_use",
        "id": "toolu_01Abc123",
        "name": "Bash",
        "input": {
          "command": "cd /Users/alice/work/myapp && npm test",
          "description": "Run test suite"
        }
      }
    ],
    "stop_reason": "tool_use"
  }

  ─── Claude Code processes this ──────────────────────────────

  1. Extracts: name="Bash", id="toolu_01Abc123", input={command: "cd... && npm test"}
  2. Fires PreToolUse hook with JSON payload
  3. Checks permission rules
  4. Shows to user: "Run test suite: npm test" [allow/deny prompt or auto]
  5. Executes: spawns shell process, runs command
  6. Captures stdout/stderr/exit code
  7. Fires PostToolUse hook

  ─── Injected back into messages ─────────────────────────────

  {
    "role": "user",
    "content": [
      {
        "type": "tool_result",
        "tool_use_id": "toolu_01Abc123",
        "content": "PASS src/auth/token.test.ts\nPASS src/auth/refresh.test.ts\n\nTest Suites: 2 passed, 2 total\nTests:       14 passed, 14 total\nTime:        2.341s"
      }
    ]
  }

  ─── Next API call includes this result ──────────────────────
  Claude sees test results, decides next action
```

---

## Tool Error Handling

When a tool fails, `is_error: true` tells Claude it went wrong:

```json
{
  "role": "user",
  "content": [
    {
      "type": "tool_result",
      "tool_use_id": "toolu_01Abc123",
      "is_error": true,
      "content": "Command failed with exit code 1:\nnpm ERR! missing script: test"
    }
  ]
}
```

Claude receives this and typically:
- Diagnoses the error
- Tries an alternative (e.g., `npx jest` instead of `npm test`)
- Reports to user if unrecoverable

---

## How the Edit Tool Works Internally

Edit is the most complex tool because it requires exact string matching.

```
  Claude calls Edit with:
  {
    "file_path": "src/auth/token.ts",
    "old_string": "  expiresIn: '1h'\n  })\n}",
    "new_string": "  expiresIn: '24h'\n  })\n}"
  }

  Claude Code:
  1. Reads file into memory
  2. Searches for EXACT match of old_string (including whitespace)
  3. If not found exactly: tool_result with is_error: true
     "Error: old_string not found. The file may have changed."
  4. If found: replaces with new_string
  5. Writes file back to disk
  6. Returns success with diff preview
```

This is why Claude must Read a file before Editing it — it needs to know
the exact current content to construct old_string correctly.

```
  Common failure pattern:
    Claude tries to edit without reading first
    → old_string doesn't match exactly
    → Error: old_string not found
    → Claude reads the file
    → Claude retries Edit with correct old_string

  Better pattern:
    Claude reads file
    Claude constructs old_string from actual file content
    Edit succeeds first try
```

---

## How Grep Works Under the Hood

Grep uses ripgrep (rg) as the underlying engine:

```
  Claude calls:
  {
    "pattern": "export function.*Token",
    "path": "src/",
    "type": "ts",
    "output_mode": "content",
    "-n": true
  }

  Claude Code translates to:
  rg --type ts -n "export function.*Token" src/

  Returns:
  src/auth/token.ts:3:export function generateToken(userId: string): string {
  src/auth/refresh.ts:7:export function refreshToken(token: string): string {
```

---

## MCP Tool Calls — Different Wire Protocol

When Claude uses an MCP tool, it goes through a different path:

```
  Claude calls: mcp__github__search_repositories
  {
    "type": "tool_use",
    "name": "mcp__github__search_repositories",
    "input": {
      "query": "claude code auth",
      "language": "typescript"
    }
  }

  Claude Code:
  1. Parses tool name: server=github, tool=search_repositories
  2. Finds GitHub MCP server process (already running or spawns it)
  3. Sends JSON-RPC over stdio:
     {
       "jsonrpc": "2.0",
       "method": "tools/call",
       "params": {
         "name": "search_repositories",
         "arguments": {
           "query": "claude code auth",
           "language": "typescript"
         }
       },
       "id": 1
     }
  4. MCP server responds with JSON-RPC response
  5. Result injected as tool_result block back to Claude
```

---

## Tool_Use ID — Why It Matters

Every tool_use has a unique ID (`toolu_01...`). This ID is used to:

1. Match results back to the correct call (parallel tool calls)
2. Track which tools were used in a session
3. Reference in hooks (the hook payload includes the tool_use_id)
4. Resume interrupted tool calls

```
  Parallel call example:

  Assistant:
    tool_use id="toolu_01AAA" name="Read" input={file: "a.ts"}
    tool_use id="toolu_01BBB" name="Read" input={file: "b.ts"}
    tool_use id="toolu_01CCC" name="Bash" input={command: "ls"}

  Claude Code: all three execute simultaneously

  User:
    tool_result tool_use_id="toolu_01AAA" content="...a.ts..."
    tool_result tool_use_id="toolu_01BBB" content="...b.ts..."
    tool_result tool_use_id="toolu_01CCC" content="...ls output..."

  IDs ensure Claude knows which result belongs to which call.
```

---

## The Read Tool — Line Numbers

Read returns file content with cat -n format:

```
  Input:  { "file_path": "src/auth/token.ts", "offset": 1, "limit": 10 }

  Output:
  "     1\timport jwt from 'jsonwebtoken'\n     2\t\n     3\texport function generateToken..."

  Format: [spaces][line number][tab][content]

  Why line numbers?
  Claude uses line numbers to construct precise old_string values for Edit.
  "Replace line 4-6" → Claude reads those exact lines → builds old_string.
```

---

## Tool Call Chain — Typical Bug Fix Session

```
  User: "Fix the token expiry bug"

  Turn 1:  Bash("grep -r 'expiresIn' src/")     → find relevant files
  Turn 2:  Read("src/auth/token.ts")             → read the file
  Turn 3:  Read("src/auth/types.ts")             → read related types
  Turn 4:  Edit("src/auth/token.ts", ...)        → fix the bug
  Turn 5:  Bash("npm test -- auth")              → verify fix
  Turn 6:  (end_turn)                             → report to user

  Each turn = 1 API call
  Full conversation history included in every call
  Total API calls for this task: 6
```

---

## Summary

```
┌────────────────────────────────────────────────────────────────┐
│  TOOL PROTOCOL CHEATSHEET                                      │
├────────────────────────────────────────────────────────────────┤
│  Format         : tool_use block in assistant message         │
│  Result format  : tool_result block in user message           │
│  Matching       : tool_use_id links call to result            │
│  Error signal   : is_error: true in tool_result               │
│  Parallel       : multiple tool_use blocks in one response    │
│  MCP tools      : routed via JSON-RPC to server process       │
│  Edit requires  : exact old_string match (read first!)        │
│  Read format    : line numbers in cat -n format               │
│  Grep engine    : ripgrep (rg) under the hood                 │
│  Hook intercept : between tool selection and execution        │
└────────────────────────────────────────────────────────────────┘
```
