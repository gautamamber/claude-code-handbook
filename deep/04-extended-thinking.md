# Deep 04 — Extended Thinking: How Claude Reasons Internally

---

## What Is Extended Thinking?

Extended thinking gives Claude dedicated space to reason step-by-step
before producing its final response. The thinking is visible to you
but is fundamentally different from the normal response.

```
  Normal mode:
  ┌──────────────────────────────────────────────────────┐
  │  API call → Claude responds → done                   │
  │  Thinking: hidden (internal, not transmitted)        │
  └──────────────────────────────────────────────────────┘

  Extended thinking mode:
  ┌──────────────────────────────────────────────────────┐
  │  API call → Claude THINKS (visible) → Claude responds│
  │                                                      │
  │  <thinking>                                          │
  │    Let me break down this problem...                 │
  │    First, I need to understand the data structure... │
  │    Option A would work but has O(n²) complexity...   │
  │    Option B is better because...                     │
  │  </thinking>                                         │
  │                                                      │
  │  [final response after reasoning]                    │
  └──────────────────────────────────────────────────────┘
```

---

## How It Works in the API

Extended thinking requires a specific API configuration:

```json
POST /v1/messages
{
  "model": "claude-sonnet-4-6",
  "max_tokens": 16000,
  "thinking": {
    "type": "enabled",
    "budget_tokens": 10000
  },
  "messages": [...]
}
```

`budget_tokens` = how many tokens Claude can use for thinking (1,024 minimum).
Claude decides how much of the budget to actually use.

---

## The Response Structure

When thinking is enabled, the response has a new content block type:

```json
{
  "role": "assistant",
  "content": [
    {
      "type": "thinking",
      "thinking": "I need to analyze this auth bug carefully.\n\nLooking at the token.ts file, I can see the JWT is signed with process.env.JWT_SECRET. The expiry is set to '1h'.\n\nThe bug report says tokens aren't expiring. Let me think about what could cause this...\n\nPossibility 1: JWT_SECRET is undefined, causing jwt.sign to fail silently...\nPossibility 2: The expiresIn option is being ignored...\nPossibility 3: Clock skew between servers...\n\nThe most likely cause is..."
    },
    {
      "type": "text",
      "text": "I've identified the issue. The JWT secret is missing from the environment, which causes jsonwebtoken to use an empty string as the secret. Here's the fix..."
    }
  ]
}
```

Two blocks: `thinking` (reasoning) + `text` (final answer).

---

## Extended Thinking and Context Window

This is the key cost to understand:

```
  Normal 10-turn session:
  Each response: ~500 tokens average
  Total response tokens: 5,000

  Extended thinking 10-turn session (budget: 10,000/turn):
  Each response: ~500 tokens (text) + up to 10,000 tokens (thinking)
  Total response tokens: up to 105,000

  IMPACT: Extended thinking can fill the context window 20x faster
```

```
  Context consumption visualization:

  Normal:      ██░░░░░░░░░░░░░░░░░░   10% after 10 turns
  Thinking:    ████████████████████   100% after same 10 turns (full!)
```

---

## Thinking Blocks in Tool Call Loops

When using extended thinking with tools, the thinking persists across turns:

```
  Turn 1:
    [thinking block]  ← Claude's reasoning about approach
    [tool_use: Read file]

  Turn 2 (after tool result):
    [thinking block]  ← Claude's reasoning about what it read
    [tool_use: Edit file]

  Turn 3:
    [thinking block]  ← Claude's reasoning about the edit
    [text: "Done"]

  Each turn's thinking is preserved in the messages array.
  This compounds the context window usage.
```

---

## Streaming with Extended Thinking

Thinking streams BEFORE the final response text:

```
  event: content_block_start
  data: {"type": "content_block_start", "content_block": {"type": "thinking", "thinking": ""}}

  event: content_block_delta
  data: {"delta": {"type": "thinking_delta", "thinking": "Let me think about"}}

  event: content_block_delta
  data: {"delta": {"type": "thinking_delta", "thinking": " this carefully..."}}

  ... (thinking streams as it's generated) ...

  event: content_block_stop   ← thinking complete

  event: content_block_start
  data: {"content_block": {"type": "text", "text": ""}}

  event: content_block_delta
  data: {"delta": {"type": "text_delta", "text": "Here's the solution..."}}
```

You see Claude "thinking" in real-time before its answer appears.

---

## When Claude Code Uses Extended Thinking

Claude Code doesn't always use extended thinking. It's model/config dependent:

```
  Conditions for extended thinking:
  • Model supports it (Sonnet 4.6, Opus 4.6)
  • thinking.type = "enabled" in API config
  • budget_tokens > 0

  When it's most valuable:
  • Complex multi-step reasoning (algorithm design)
  • Debugging subtle issues
  • Architecture decisions
  • Math/logic problems
  • Tasks where showing reasoning builds trust
```

---

## Interleaved Thinking with Tools

Extended thinking with tool use enables a powerful pattern:

```
  User: "Why is the authentication failing?"

  Turn 1:
    THINKING: "The user reports auth failures. I need to:
    1. Find the auth middleware
    2. Check error logs
    3. Look at the JWT verification logic
    Let me start with finding the middleware..."

    TOOL: Glob("**/middleware/auth*")

  Turn 2:
    THINKING: "Found auth.ts. The middleware uses verifyToken.
    Let me read the implementation. I suspect the issue is in
    how it handles expired tokens vs invalid tokens..."

    TOOL: Read("src/middleware/auth.ts")

  Turn 3:
    THINKING: "I can see the bug now. On line 23, the catch block
    swallows both TokenExpiredError and JsonWebTokenError and returns
    the same 401. But the client expects different responses.
    Actually wait — the real issue is line 18: req.user = decoded
    is called BEFORE awaiting the verification promise.
    This means req.user might be set with unverified data..."

    TEXT: "Found the bug. On line 18 of auth.ts, req.user is
    assigned before the JWT verification completes..."
```

The thinking trace shows you exactly how Claude reasoned to its conclusion.

---

## budget_tokens — What It Controls

```
  budget_tokens: 1024     ← minimum, very short thinking
  budget_tokens: 5000     ← typical for moderate complexity
  budget_tokens: 10000    ← for complex reasoning tasks
  budget_tokens: 32000    ← for very complex multi-step problems

  Claude doesn't always use the full budget.
  For simple tasks, it may use 200 tokens even with budget=10000.
  For complex tasks, it fills the budget if needed.
```

---

## Encrypted Thinking (API Feature)

There's also "encrypted thinking" mode:

```json
{
  "thinking": {
    "type": "enabled",
    "budget_tokens": 5000,
    "encrypted": true
  }
}
```

In this mode:
- The thinking block content is encrypted in the response
- You cannot read the thinking
- The encrypted thinking IS included in the next API call
- Claude can access its own previous thinking to build on it
- Used for: multi-turn reasoning continuity without exposing internals

---

## Summary

```
┌────────────────────────────────────────────────────────────────┐
│  EXTENDED THINKING CHEATSHEET                                  │
├────────────────────────────────────────────────────────────────┤
│  Config         : thinking: {type: "enabled", budget_tokens: N}│
│  Min budget     : 1,024 tokens                                 │
│  Response block : type: "thinking" + type: "text"             │
│  Context cost   : up to 20x more than normal mode             │
│  Streaming      : thinking streams before text                 │
│  Tool loops     : thinking persists across all turns           │
│  Encrypted mode : thinking hidden but reusable across turns    │
│  Best for       : debugging, architecture, complex reasoning   │
│  Avoid for      : simple tasks (wastes context budget)         │
└────────────────────────────────────────────────────────────────┘
```
