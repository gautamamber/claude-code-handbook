# Expert 06 — Cost Optimization: Reducing Token Usage

---

## Understanding What Costs Money

Every token sent to AND received from the API costs money.
In Claude Code, tokens accumulate in 3 ways:

```
  ┌──────────────────────────────────────────────────────────────┐
  │  COST SOURCES                                                │
  │                                                              │
  │  1. INPUT TOKENS (per API call)                             │
  │     • System prompt (~2,000 tokens) — every call            │
  │     • CLAUDE.md content — every call                        │
  │     • Full conversation history — every call (grows!)       │
  │     • All tool results from prior turns — every call        │
  │     • File contents that were read — stays in history       │
  │                                                              │
  │  2. OUTPUT TOKENS                                           │
  │     • Claude's responses                                     │
  │     • Claude's reasoning (if thinking enabled)              │
  │     • Tool call JSON                                         │
  │                                                              │
  │  3. THINKING TOKENS (most expensive output)                 │
  │     • Extended thinking = up to 50k tokens per turn         │
  │     • Billed at output token rate                           │
  └──────────────────────────────────────────────────────────────┘
```

---

## Model Cost Comparison

```
  ┌──────────────────┬─────────────┬──────────────┬────────────────────┐
  │  Model           │  Input /1M  │  Output /1M  │  Best Use          │
  ├──────────────────┼─────────────┼──────────────┼────────────────────┤
  │  Haiku 3.5       │    $0.80    │    $4.00     │  Exploration       │
  │  Sonnet 4.6      │    $3.00    │   $15.00     │  Most coding       │
  │  Opus 4.6        │   $15.00    │   $75.00     │  Complex reasoning │
  └──────────────────┴─────────────┴──────────────┴────────────────────┘

  Cost example: 100k token conversation
  Haiku:  $0.08 input + $0.40 output = $0.48
  Sonnet: $0.30 input + $1.50 output = $1.80
  Opus:   $1.50 input + $7.50 output = $9.00

  Haiku is 18x cheaper than Opus for the same conversation.
```

---

## Strategy 1: Right Model for the Job

```
  Task                              Best Model    Why
  ────────────────────────────────────────────────────────────
  Explore/search codebase           Haiku         Simple lookups
  Write documentation               Haiku         Not complex
  Simple bug fix (1-2 files)        Haiku         Straightforward
  Standard feature implementation   Sonnet        Balanced
  Complex multi-file refactor       Sonnet        Good reasoning
  Architecture design               Opus          Deep reasoning
  Security audit                    Sonnet        Good + cheap
  Debugging mysterious bug          Sonnet/Opus   Depends on depth

  Rule of thumb:
  Start with Haiku → if it fails, try Sonnet → if still fails, Opus
```

---

## Strategy 2: Control Context Size

The #1 cost driver is context size × number of turns.
Large context + many turns = expensive.

```
  BAD pattern:
    Turn 1: Read 10 large files (50k tokens)
    Turn 2: Read 5 more files   (25k tokens)
    Context: 75k tokens → expensive for every subsequent call

  GOOD pattern:
    Turn 1: Grep for relevant function (1k tokens)
    Turn 2: Read only that function (2k tokens)
    Context: 3k tokens → cheap for every subsequent call
```

**Practical techniques:**

```bash
# Instead of reading entire files:
"Read only the generateToken function in src/auth/token.ts"

# Instead of running verbose commands:
Bash("npm test 2>&1 | tail -20")  # Last 20 lines only
Bash("npm test 2>&1 | grep -E 'FAIL|Error|PASS' | head -30")

# Instead of grepping entire codebase:
"Search for generateToken in src/auth/ only"
```

---

## Strategy 3: Use Subagents for Heavy Research

Subagents have their OWN context window. They don't pollute the parent.

```
  WITHOUT subagent:
    Parent reads 20 files → 40k tokens in parent context
    Every subsequent turn: 40k tokens re-sent to API

  WITH subagent:
    Spawn subagent (haiku): "read 20 files, summarize findings"
    Subagent uses its own 200k window for all reads
    Returns 2k token summary to parent
    Parent context grows by 2k, not 40k

  Savings: (40k - 2k) × N_remaining_turns × token_rate
```

```yaml
# Cheap explorer subagent
---
name: explorer
model: haiku
tools: Read, Grep, Glob
maxTurns: 15
effort: low
---
Find and summarize. Be concise. Return structured data.
```

---

## Strategy 4: Disable Extended Thinking When Not Needed

Extended thinking = up to 50k output tokens PER TURN.
For a 10-turn session: up to 500k thinking tokens.

```
  Task type                         Need thinking?
  ────────────────────────────────────────────────
  Read a file and fix a typo        NO
  Run npm test and report           NO
  Write a README                    NO
  Design a complex algorithm        YES
  Debug mysterious race condition   MAYBE
  Refactor module architecture      MAYBE

  Disable:  Meta+T toggle in session
            OR set in /config
            OR export MAX_THINKING_TOKENS=0
            OR effort: low in agent frontmatter
```

---

## Strategy 5: Use /compact Proactively

Don't wait for auto-compaction. Compact when you naturally finish a phase.

```
  Workflow:
    Phase 1: Research    → lots of file reads
    /compact "keep: auth module structure, JWT expiry bug"
    Phase 2: Fix         → edits with clean context

  vs waiting:
    Phase 1: Research    → file reads fill context
    Phase 2: Fix         → still paying for all those file reads
    [auto-compact at 95%] → random point, loses some context
```

```bash
# Targeted compaction
/compact focus on the auth bug and the token.ts fix

# General compaction
/compact

# Start fresh between unrelated tasks
/clear
```

---

## Strategy 6: Prompt Caching

Prompt caching gives 90% discount on re-used input tokens.
The system prompt + CLAUDE.md is typically cached after the first call.

```
  Session start (no cache):
  System prompt (2k tokens) + CLAUDE.md (2k tokens) = 4k tokens
  Cost: 4k × full rate

  Turn 2, 3, 4... (cache hit):
  Same 4k tokens = 4k × 10% of rate  ← 90% savings

  A 20-turn session:
  Without cache: 20 × 4k = 80k tokens at full rate
  With cache:    4k full + 19 × 4k × 10% = 4k + 7.6k = 11.6k effective
  Savings: 85% on system prompt tokens
```

Caching happens automatically. Your job: keep CLAUDE.md stable.
Every change to CLAUDE.md invalidates the cache.

---

## Strategy 7: Batch Related Work

Context "warms up" across turns. Starting fresh for each small task wastes the warm-up cost.

```
  EXPENSIVE:
    Session 1: "Fix auth bug"     → 5 turns to understand codebase
    Session 2: "Fix API bug"      → 5 turns to re-understand codebase
    Session 3: "Fix DB bug"       → 5 turns to re-understand codebase

    Total: 15 turns × context building overhead

  CHEAPER:
    Session 1: "Fix auth bug, then API bug, then DB bug"
    → 5 turns to understand codebase
    → 3 turns for each fix (codebase already known)
    Total: 5 + 3 + 3 = 11 turns, lower overhead
```

---

## Strategy 8: Use --max-turns in CI

Runaway loops are the #1 CI cost issue. Always set limits.

```bash
# CI: strict limits
claude -p "task" \
  --max-turns 10 \
  --max-budget-usd 2.00 \
  --model haiku

# If task genuinely needs more turns: bump the limit explicitly
claude -p "complex refactor" \
  --max-turns 50 \
  --max-budget-usd 10.00 \
  --model sonnet
```

---

## Daily Cost Benchmarks

```
  Light user (1-2 hours of coding):        $1-3/day
  Regular developer (4-6 hours):           $4-8/day
  Heavy user (8+ hours):                   $10-20/day
  CI/CD pipelines (100 runs/day):          $5-50/day depending on task

  Cost per task type (approximate):
  • Quick question/fix:         $0.01-0.05
  • Feature implementation:     $0.20-1.00
  • Full module refactor:        $1.00-5.00
  • Codebase analysis:           $0.50-3.00
  • PR review:                   $0.10-0.50
```

---

## Cost Monitoring

```bash
# Check cost during session
/cost

# Check cost in CI output
claude -p "task" --output-format json | jq '{cost: .cost_usd, tokens: .usage}'

# Monthly spend: check Claude.ai billing dashboard
# Team spend: check Anthropic Console → Usage

# Set spending alert (Console feature)
# Notify when daily spend exceeds threshold
```

---

## Summary: 10x Cost Reduction Checklist

```
  □  Use Haiku for all exploration and simple tasks
  □  Use Sonnet for implementation (not Opus)
  □  Disable extended thinking for non-reasoning tasks
  □  Spawn Haiku subagents for file-heavy research
  □  Set maxTurns on all agents and CI calls
  □  Set --max-budget-usd on all CI pipeline calls
  □  Use targeted Grep/Glob instead of reading full files
  □  Pipe verbose Bash output through tail/grep
  □  Use /compact after heavy research phases
  □  Batch related tasks in one session vs many sessions

  Potential savings: 5-10x reduction in token costs
```

```
┌────────────────────────────────────────────────────────────────┐
│  COST OPTIMIZATION CHEATSHEET                                  │
├────────────────────────────────────────────────────────────────┤
│  Cheapest model  : Haiku (18x cheaper than Opus)               │
│  Biggest cost    : large context × many turns                  │
│  Context killer  : reading large files repeatedly              │
│  Thinking cost   : up to 50k tokens/turn = very expensive      │
│  Cache discount  : 90% off repeated system prompt tokens       │
│  Subagent trick  : offload file reads → only summary returns   │
│  CI protection   : --max-turns + --max-budget-usd              │
│  Monitor         : /cost in session, Console for team          │
└────────────────────────────────────────────────────────────────┘
```
