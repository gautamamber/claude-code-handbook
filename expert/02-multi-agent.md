# Expert 02 — Multi-Agent Orchestration Patterns

---

## Why Multi-Agent?

One agent has one context window. Complex tasks exceed what one context can hold.
Multi-agent systems distribute work across isolated contexts.

```
  Single agent:
  ┌─────────────────────────────────────────────────────┐
  │  Context window: 200,000 tokens                     │
  │  Task: "Review entire 50,000-line codebase"         │
  │                                                     │
  │  ██████████ history                                 │
  │  ██████████████████ file reads (fills fast)         │
  │  ░░░░░░░░░ out of space after 20 files              │
  └─────────────────────────────────────────────────────┘

  Multi-agent:
  ┌──────────────────────────────────────────────────────────────┐
  │  Orchestrator (10k tokens used)                              │
  │   ├── Agent A: reviews src/auth/  → 5k token summary        │
  │   ├── Agent B: reviews src/api/   → 5k token summary        │
  │   └── Agent C: reviews src/db/    → 5k token summary        │
  │                                                              │
  │  Each agent uses its own full 200k window for its module.   │
  │  Orchestrator sees compact summaries, not raw file contents. │
  └──────────────────────────────────────────────────────────────┘
```

---

## Pattern 1: Sequential Chaining

One agent's output feeds the next agent's input.

```
  ┌──────────────┐
  │  User prompt │
  └──────┬───────┘
         │
         ▼
  ┌──────────────┐    produces analysis
  │  Agent: Plan │ ──────────────────────┐
  └──────────────┘                       │
                                         ▼
                                  ┌──────────────┐    writes code
                                  │  Agent: Code │ ─────────────┐
                                  └──────────────┘              │
                                                                 ▼
                                                          ┌──────────────┐
                                                          │ Agent: Tests │
                                                          └──────────────┘
```

**Implementation:**

```yaml
# .claude/agents/full-feature.md
---
name: full-feature
description: "Build a complete feature: plan → implement → test"
tools: Agent(planner, implementer, tester), Read, Bash
model: sonnet
---

To build a feature:
1. Spawn planner agent to design the approach
2. Pass plan to implementer agent to write code
3. Pass implementation to tester agent to write tests
4. Report summary of all three phases

Always run agents in sequence, passing output of each to the next.
```

---

## Pattern 2: Parallel Research

Multiple agents research independently, orchestrator synthesizes.

```
  Task: "Understand how authentication works in this codebase"
                │
                ▼
  ┌─────────────────────────────────────────────────────┐
  │  Orchestrator spawns 3 parallel subagents           │
  │                                                     │
  │  Agent A (background): Find all auth middleware     │
  │  Agent B (background): Find all JWT usage           │
  │  Agent C (background): Find all session handling    │
  │                                                     │
  │  All 3 run simultaneously                          │
  └──────────────────────────────────────────────────────┘
         │             │             │
         ▼             ▼             ▼
    [summary A]   [summary B]   [summary C]
         │             │             │
         └─────────────┴─────────────┘
                       │
                       ▼
         Orchestrator synthesizes findings
```

**Implementation in session:**

```
You: "Understand how auth works, run agents in parallel"
Claude: I'll spawn three parallel explorers...
[Uses Agent tool 3 times with run_in_background: true]
[Waits for all three]
[Synthesizes summaries]
```

---

## Pattern 3: Specialist Pipeline

Each stage has a domain expert with restricted access.

```
  ┌───────────────────────────────────────────────────────────────┐
  │  PIPELINE                                                     │
  │                                                               │
  │  Input ──► [Security Auditor]  ──► Security report           │
  │        ──► [Performance Audit] ──► Perf report               │
  │        ──► [Code Quality]      ──► Quality report            │
  │                   │                                           │
  │                   ▼                                           │
  │             [Orchestrator]                                    │
  │             Combines reports into final PR review             │
  └───────────────────────────────────────────────────────────────┘
```

**Agents for this pipeline:**

```markdown
# .claude/agents/security-auditor.md
---
name: security-auditor
tools: Read, Grep, Glob
model: sonnet
maxTurns: 20
---
Find security vulnerabilities only. Return structured report.
```

```markdown
# .claude/agents/perf-auditor.md
---
name: perf-auditor
tools: Read, Grep, Glob, Bash
model: sonnet
maxTurns: 15
---
Analyze performance: O(n) complexity, N+1 queries, memory leaks.
```

```markdown
# .claude/agents/pr-orchestrator.md
---
name: pr-orchestrator
tools: Agent(security-auditor, perf-auditor, code-quality), Read
model: opus
maxTurns: 50
---
For every PR review:
1. Run security-auditor, perf-auditor, code-quality in parallel
2. Collect all reports
3. Combine into unified PR review with severity rankings
```

---

## Pattern 4: Agent Teams (Experimental)

Agent teams allow multiple agents to work concurrently with shared visibility.

```
  Enable: CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1

  ┌──────────────────────────────────────────────────────┐
  │  Team Lead (you)                                     │
  │   │                                                  │
  │   ├── Teammate A: "Work on auth module"              │
  │   ├── Teammate B: "Work on API module"               │
  │   └── Teammate C: "Work on tests"                   │
  │                                                      │
  │  Each teammate has FULL context window.              │
  │  Each can see shared conversation.                   │
  │  ~7x token cost vs single agent.                    │
  └──────────────────────────────────────────────────────┘
```

Use teams for tightly-coupled work where agents need to coordinate.
Use parallel subagents for independent work (cheaper).

---

## Pattern 5: Supervisor + Workers

Supervisor decides work allocation. Workers execute. Supervisor reviews.

```
  ┌──────────────────────────────────────────────────────────────┐
  │                    SUPERVISOR                                │
  │                                                              │
  │  Receives task → breaks into subtasks                        │
  │  Assigns each subtask to specialist worker                  │
  │  Reviews worker output                                       │
  │  Merges results                                              │
  │  Reports to user                                             │
  └──────────────────────┬───────────────────────────────────────┘
                         │  spawns workers
             ┌───────────┼───────────┐
             ▼           ▼           ▼
        [Worker A]  [Worker B]  [Worker C]
        (haiku)     (haiku)     (haiku)
        auth task   api task    db task
```

**Why haiku workers?**
- Haiku is 15-20x cheaper than Opus
- Workers do specific, bounded tasks
- Supervisor (Opus or Sonnet) does reasoning
- Dramatic cost savings with same quality

---

## Pattern 6: Iterative Refinement

Agent produces output → Reviewer critiques → Agent improves → repeat.

```
  ┌──────────┐
  │  Draft   │ ← Agent A writes first version
  └────┬─────┘
       │
       ▼
  ┌──────────┐
  │  Review  │ ← Agent B critiques (security, quality, etc.)
  └────┬─────┘
       │
       ▼
  ┌──────────┐
  │  Revise  │ ← Agent A incorporates feedback
  └────┬─────┘
       │
       ▼
  Repeat N times or until reviewer approves
```

---

## Cost Analysis — Choosing the Right Pattern

```
  ┌──────────────────────────────────────────────────────────────┐
  │  PATTERN             TOKEN MULTIPLIER    BEST FOR           │
  ├──────────────────────────────────────────────────────────────┤
  │  Single agent        1x                 Simple tasks        │
  │  Sequential chain    ~2-3x              Dependent phases    │
  │  Parallel subagents  ~1.2-1.5x         Independent research│
  │  Agent teams         ~7x               Coordinated work    │
  │  Supervisor+workers  ~2-4x             Large codebases     │
  └──────────────────────────────────────────────────────────────┘

  Cost optimization tips:
  • Use Haiku for all worker/explorer agents
  • Use Sonnet for implementer agents
  • Reserve Opus for orchestrator/supervisor only
  • Set maxTurns on all subagents (prevent loops)
  • Use background: true for independent parallel work
```

---

## Orchestrator Agent Template

```markdown
---
name: project-orchestrator
description: "Coordinates complex multi-step tasks across specialist agents"
model: opus
tools: Agent(explorer, implementer, reviewer, tester), Read, Bash
maxTurns: 100
effort: high
---

You coordinate complex software engineering tasks.

When given a task:
1. Break it into independent subtasks
2. Identify which specialist handles each subtask
3. Determine which can run in parallel vs must be sequential
4. Spawn agents with clear, bounded prompts
5. Collect results and synthesize
6. Report final summary to user

Specialists available:
- explorer: Read-only codebase research (use Haiku, fast)
- implementer: Code changes (use Sonnet, careful)
- reviewer: Code review without changes (use Sonnet)
- tester: Write and run tests (use Sonnet)

Always run independent tasks in parallel (run_in_background: true).
Always set clear output expectations in each agent's prompt.
```

---

## Writing Good Agent Prompts

When spawning a subagent, the prompt is all it gets. Make it self-contained.

```
  BAD prompt (too vague):
  "Fix the auth bug"
  → Subagent doesn't know which file, what bug, what "fixed" means

  GOOD prompt (self-contained):
  "In src/auth/token.ts, the generateToken function uses 'expiresIn: 1h'.
  Change it to '24h'. Run the auth tests (npm test -- auth) to verify.
  Report: what you changed, test results, any issues found."
  → Subagent knows exactly what to do, what to check, what to report
```

Key elements of a good subagent prompt:
1. Specific file paths
2. Exact expected behavior
3. How to verify success
4. What to return in the summary

---

## Subagent Communication Constraints

```
  Parent                          Subagent
  ──────────────────────────────────────────
  Sends: explicit prompt text       ✓
  Sends: conversation history       ✗  (not seen by subagent)
  Sends: current tool results       ✗  (not seen by subagent)
  Sends: edited file contents       ✗  (must re-read if needed)

  Receives: final response text     ✓
  Receives: intermediate steps      ✗  (not visible)
  Receives: files created           via disk (shared filesystem)

  Workaround for sharing state:
  → Write summary to a temp file before subagent ends
  → Subagent returns structured JSON in its final response
  → Parent parses JSON from tool_result
```

---

## Summary

```
┌────────────────────────────────────────────────────────────────┐
│  MULTI-AGENT CHEATSHEET                                        │
├────────────────────────────────────────────────────────────────┤
│  Sequential     : pass output of A as input to B               │
│  Parallel       : run_in_background: true for independent work │
│  Teams          : CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1       │
│  Cost model     : Haiku workers + Sonnet/Opus orchestrator     │
│  Subagent limit : cannot spawn sub-subagents (1 level only)    │
│  Context wall   : subagents don't see parent history           │
│  Data sharing   : via filesystem or structured prompt return   │
│  maxTurns       : always set on workers to prevent loops       │
│  Best pattern   : supervisor(opus) + workers(haiku)            │
└────────────────────────────────────────────────────────────────┘
```
