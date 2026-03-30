# Deep 01 — The Agentic Loop: How Claude Code Actually Runs

---

## The Core Insight

Claude Code is not a chatbot that happens to run commands.
It is an **agentic loop** — a program that repeatedly calls the Claude API
until a task is fully complete.

Most users see the final result. This document shows every step.

---

## The Agentic Loop — Full Cycle

```
  USER TYPES A MESSAGE
          │
          ▼
  ┌─────────────────────────────────────────────────────────────┐
  │  STEP 1: Build Context                                      │
  │                                                             │
  │  Assemble the messages array:                               │
  │  • system prompt                                            │
  │  • CLAUDE.md content                                        │
  │  • memory (MEMORY.md)                                       │
  │  • full conversation history                                │
  │  • new user message                                         │
  └──────────────────────────┬──────────────────────────────────┘
                             │
                             ▼
  ┌─────────────────────────────────────────────────────────────┐
  │  STEP 2: API Call                                           │
  │                                                             │
  │  POST /v1/messages                                          │
  │  {                                                          │
  │    model: "claude-sonnet-4-6",                              │
  │    messages: [...full history...],                          │
  │    tools: [...all tool definitions...],                     │
  │    stream: true                                             │
  │  }                                                          │
  └──────────────────────────┬──────────────────────────────────┘
                             │
                             ▼
  ┌─────────────────────────────────────────────────────────────┐
  │  STEP 3: Receive Response (streaming)                       │
  │                                                             │
  │  Two possible stop_reason values:                           │
  │                                                             │
  │  stop_reason: "end_turn"    ← Claude is done, no tools     │
  │  stop_reason: "tool_use"    ← Claude wants to use a tool   │
  └──────────────────────────┬──────────────────────────────────┘
                             │
              ┌──────────────┴──────────────┐
              │                             │
         end_turn                       tool_use
              │                             │
              ▼                             ▼
  ┌────────────────────┐     ┌──────────────────────────────────┐
  │  DONE              │     │  STEP 4: Execute Tool            │
  │  Show response     │     │                                  │
  │  to user           │     │  1. PreToolUse hook fires        │
  └────────────────────┘     │  2. Permission check             │
                             │  3. Tool runs (Bash/Read/Edit)   │
                             │  4. PostToolUse hook fires       │
                             │  5. Result captured              │
                             └──────────────────┬───────────────┘
                                                │
                                                ▼
                             ┌──────────────────────────────────┐
                             │  STEP 5: Inject Tool Result      │
                             │                                  │
                             │  Append to messages array:       │
                             │  - Claude's tool_use block       │
                             │  - Our tool_result block         │
                             └──────────────────┬───────────────┘
                                                │
                                                ▼
                                       ← LOOP BACK TO STEP 2 →
```

---

## How the Loop Decides to Stop

Claude Code does NOT have a fixed iteration limit exposed publicly.
The loop stops when Claude returns `stop_reason: "end_turn"`.

Claude decides to stop when:
- The task is complete
- It cannot proceed without user input
- It hits an error it cannot recover from
- It has used the allowed tools and gathered enough information

```
  Loop iteration tracking (simplified):

  Turn 1: user message → API → tool_use (Read file)
  Turn 2: tool result  → API → tool_use (Grep for pattern)
  Turn 3: tool result  → API → tool_use (Edit file)
  Turn 4: tool result  → API → tool_use (Bash npm test)
  Turn 5: tool result  → API → end_turn (task complete)

  Each "turn" = one API call
  Each API call includes FULL conversation history
```

---

## The API Is Stateless

This is critical to understand:

```
  The Anthropic Messages API has NO session state.
  Every single API call sends the FULL conversation history.

  Turn 1:
    messages: [
      { role: "user", content: "Fix the auth bug" }
    ]

  Turn 2:
    messages: [
      { role: "user",      content: "Fix the auth bug" },
      { role: "assistant", content: [tool_use block] },
      { role: "user",      content: [tool_result block] }
    ]

  Turn 3:
    messages: [
      { role: "user",      content: "Fix the auth bug" },
      { role: "assistant", content: [tool_use block 1] },
      { role: "user",      content: [tool_result block 1] },
      { role: "assistant", content: [tool_use block 2] },
      { role: "user",      content: [tool_result block 2] }
    ]

  This keeps growing until end_turn or context limit
```

Claude Code manages this growing array. When it gets too large → compaction.

---

## The API Request Structure

Every API call Claude Code makes looks like this:

```json
POST https://api.anthropic.com/v1/messages

{
  "model": "claude-sonnet-4-6",
  "max_tokens": 8096,
  "stream": true,
  "system": "You are Claude Code...[system prompt]...",

  "tools": [
    {
      "name": "Bash",
      "description": "Execute bash commands in the terminal",
      "input_schema": {
        "type": "object",
        "properties": {
          "command": {
            "type": "string",
            "description": "The bash command to execute"
          },
          "timeout": {
            "type": "integer",
            "description": "Timeout in milliseconds"
          }
        },
        "required": ["command"]
      }
    },
    {
      "name": "Read",
      "description": "Read a file from the filesystem",
      "input_schema": { ... }
    },
    {
      "name": "Edit",
      "description": "Edit a file by replacing text",
      "input_schema": { ... }
    }
    // ... all other tools
  ],

  "messages": [
    // full conversation history here
  ]
}
```

---

## What a Tool_Use Block Looks Like

When Claude decides to run a tool, it returns this in the response:

```json
{
  "role": "assistant",
  "content": [
    {
      "type": "text",
      "text": "I'll read the auth file first to understand the current implementation."
    },
    {
      "type": "tool_use",
      "id": "toolu_01XyZ123abc",
      "name": "Read",
      "input": {
        "file_path": "/Users/alice/work/myapp/src/auth/token.ts"
      }
    }
  ]
}
```

Claude can:
- Include text BEFORE the tool call (reasoning)
- Call ONE tool at a time (standard)
- Call MULTIPLE tools in parallel (if model supports it)

---

## What a Tool_Result Block Looks Like

After executing the tool, Claude Code adds this to the conversation:

```json
{
  "role": "user",
  "content": [
    {
      "type": "tool_result",
      "tool_use_id": "toolu_01XyZ123abc",
      "content": "1  import jwt from 'jsonwebtoken'\n2  \n3  export function generateToken(userId: string): string {\n4    return jwt.sign({ userId }, process.env.JWT_SECRET!, {\n5      expiresIn: '1h'\n6    })\n7  }"
    }
  ]
}
```

The role is `"user"` — not assistant. Tool results are always injected as user messages.
This is because the API is a turn-based system: assistant → user → assistant → user...

---

## Parallel Tool Calls

Claude can call multiple tools in one response:

```json
{
  "role": "assistant",
  "content": [
    {
      "type": "text",
      "text": "Let me read multiple files at once."
    },
    {
      "type": "tool_use",
      "id": "toolu_01AAA",
      "name": "Read",
      "input": { "file_path": "src/auth/token.ts" }
    },
    {
      "type": "tool_use",
      "id": "toolu_01BBB",
      "name": "Read",
      "input": { "file_path": "src/auth/refresh.ts" }
    },
    {
      "type": "tool_use",
      "id": "toolu_01CCC",
      "name": "Bash",
      "input": { "command": "npm test -- --grep auth" }
    }
  ]
}
```

All three execute simultaneously. Results injected together:

```json
{
  "role": "user",
  "content": [
    {
      "type": "tool_result",
      "tool_use_id": "toolu_01AAA",
      "content": "...token.ts content..."
    },
    {
      "type": "tool_result",
      "tool_use_id": "toolu_01BBB",
      "content": "...refresh.ts content..."
    },
    {
      "type": "tool_result",
      "tool_use_id": "toolu_01CCC",
      "content": "...test output..."
    }
  ]
}
```

---

## Streaming — How Claude Code Handles It

Claude Code uses Server-Sent Events (SSE) streaming. The response arrives in chunks:

```
  event: message_start
  data: {"type": "message_start", "message": {"id": "msg_01...", "type": "message"}}

  event: content_block_start
  data: {"type": "content_block_start", "index": 0, "content_block": {"type": "text", "text": ""}}

  event: content_block_delta
  data: {"type": "content_block_delta", "index": 0, "delta": {"type": "text_delta", "text": "I'll"}}

  event: content_block_delta
  data: {"type": "content_block_delta", "index": 0, "delta": {"type": "text_delta", "text": " read"}}

  event: content_block_delta
  data: {"type": "content_block_delta", "index": 0, "delta": {"type": "text_delta", "text": " the file"}}

  event: content_block_stop
  data: {"type": "content_block_stop", "index": 0}

  event: content_block_start
  data: {"type": "content_block_start", "index": 1, "content_block": {"type": "tool_use", "id": "toolu_01...", "name": "Read", "input": {}}}

  event: content_block_delta
  data: {"type": "content_block_delta", "index": 1, "delta": {"type": "input_json_delta", "partial_json": "{\"file_path\":"}}

  event: content_block_delta
  data: {"type": "content_block_delta", "index": 1, "delta": {"type": "input_json_delta", "partial_json": " \"src/auth.ts\"}"}}

  event: message_delta
  data: {"type": "message_delta", "delta": {"stop_reason": "tool_use"}}

  event: message_stop
  data: {"type": "message_stop"}
```

Claude Code:
1. Streams text to terminal in real-time (you see it appear live)
2. Buffers tool_use input_json until complete
3. On `message_stop` with `stop_reason: "tool_use"` → executes tools
4. Then calls API again with results

---

## What Happens When You Press Ctrl+C

```
  Claude is streaming a response...
  User presses Ctrl+C

          │
          ▼
  Claude Code catches SIGINT signal
          │
          ▼
  Streaming connection interrupted
          │
          ▼
  Current partial response discarded
          │
          ▼
  Stop hook fires (StopFailure or Stop event)
          │
          ▼
  Session returns to idle
  Conversation history up to the interrupted turn is preserved
```

If Ctrl+C is pressed during tool execution:
- The tool process receives SIGINT
- Tool output (if any) may be partial
- Claude Code decides whether to report partial result or discard

---

## How Context Is Built Before Each API Call

```
  Before every API call, Claude Code assembles:

  messages = [
    // ─── INJECTED CONTEXT (appears as first user message) ───
    {
      "role": "user",
      "content": "<system>\n[CLAUDE.md content]\n[MEMORY.md content]\n</system>"
    },
    {
      "role": "assistant",
      "content": "Understood."
    },

    // ─── ACTUAL CONVERSATION ─────────────────────────────────
    { "role": "user",      "content": "Fix the auth bug" },
    { "role": "assistant", "content": [tool_use: Read] },
    { "role": "user",      "content": [tool_result: file content] },
    { "role": "assistant", "content": [tool_use: Edit] },
    { "role": "user",      "content": [tool_result: success] },

    // ─── PENDING TURN ─────────────────────────────────────────
    // (if this is turn N, we're about to send this)
  ]
```

> Note: The exact injection mechanism for CLAUDE.md is proprietary,
> but observable behavior shows it as early context in the conversation.

---

## The stop_reason Values

```
  "end_turn"         Claude finished naturally — task complete
  "tool_use"         Claude wants to call one or more tools
  "max_tokens"       Hit token limit mid-response (rare)
  "stop_sequence"    Hit a configured stop sequence (rare in Claude Code)
```

Claude Code only loops when `stop_reason: "tool_use"`.
Everything else terminates the current turn.

---

## Error Handling in the Loop

```
  API call fails (rate limit, network, etc.)
          │
          ▼
  Claude Code checks error type:

  ┌──────────────────────────────────────────┐
  │  Rate limit (429)                        │
  │  → Exponential backoff: 1s, 2s, 4s...   │
  │  → Retry up to N times                  │
  └──────────────────────────────────────────┘

  ┌──────────────────────────────────────────┐
  │  Server error (500, 529)                 │
  │  → Short retry delay                    │
  │  → Give up after failures               │
  │  → StopFailure hook fires               │
  └──────────────────────────────────────────┘

  ┌──────────────────────────────────────────┐
  │  Auth error (401)                        │
  │  → No retry                             │
  │  → Show auth error to user              │
  └──────────────────────────────────────────┘

  ┌──────────────────────────────────────────┐
  │  Tool execution error                    │
  │  → tool_result with is_error: true       │
  │  → Claude sees the error, decides how   │
  │    to proceed (retry, alternative, stop)│
  └──────────────────────────────────────────┘
```

### Tool Error Result Format

```json
{
  "role": "user",
  "content": [
    {
      "type": "tool_result",
      "tool_use_id": "toolu_01XyZ",
      "is_error": true,
      "content": "Error: ENOENT: no such file or directory, open '/src/missing.ts'"
    }
  ]
}
```

Claude reads this and decides: try another path? Ask user? Report error?

---

## The Agentic Loop vs Single-Turn Chat

```
  SINGLE-TURN CHAT (ChatGPT-style):
  ┌──────────┐         ┌──────────┐
  │  User    │ ──────► │  Claude  │ ──── response ──── done
  └──────────┘         └──────────┘
  One API call. Done.

  AGENTIC LOOP (Claude Code):
  ┌──────────┐         ┌──────────┐
  │  User    │ ──────► │  Claude  │ ──── needs tool ─────────────┐
  └──────────┘         └──────────┘                              │
                            ▲                                     ▼
                            │                            ┌─────────────────┐
                            │                            │  Tool executes  │
                            │                            └────────┬────────┘
                            │                                     │
                            └──────────── result injected ◄───────┘
                              (loop continues until end_turn)
```

---

## Summary

```
┌────────────────────────────────────────────────────────────────┐
│  AGENTIC LOOP CHEATSHEET                                       │
├────────────────────────────────────────────────────────────────┤
│  API type       : Stateless Messages API                       │
│  Loop trigger   : stop_reason == "tool_use"                    │
│  Loop stops     : stop_reason == "end_turn"                    │
│  History        : Full conversation sent every API call        │
│  Tool call type : tool_use content block in assistant message  │
│  Tool result    : tool_result content block in user message    │
│  Parallel tools : Multiple tool_use blocks in one response     │
│  Streaming      : SSE (Server-Sent Events) for live output     │
│  Error inject   : is_error: true in tool_result                │
│  Ctrl+C         : SIGINT → interrupt stream → return to idle   │
└────────────────────────────────────────────────────────────────┘
```
