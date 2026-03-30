# Expert 01 — Custom Agents: Designing Specialized AI Workers

---

## Agents vs Skills vs Subagents — The Distinction

Most people confuse these. Here's the exact difference:

```
  ┌────────────────────────────────────────────────────────────────┐
  │  SKILL  (.claude/skills/*/SKILL.md)                           │
  │                                                                │
  │  A reusable prompt template invoked on demand.                │
  │  Runs inside the current session's context.                   │
  │  User triggers: /skill-name                                   │
  │  Claude triggers: sees description, auto-invokes              │
  │                                                                │
  │  Use when: you want a reusable task pattern                   │
  └────────────────────────────────────────────────────────────────┘

  ┌────────────────────────────────────────────────────────────────┐
  │  AGENT  (.claude/agents/*.md)                                 │
  │                                                                │
  │  A specialist Claude with a different system prompt.          │
  │  Replaces or augments the default system prompt.             │
  │  Can be the main session thread (--agent flag)                │
  │  OR spawned as an isolated subagent by the parent             │
  │                                                                │
  │  Use when: you want a completely different AI persona/role    │
  └────────────────────────────────────────────────────────────────┘

  ┌────────────────────────────────────────────────────────────────┐
  │  SUBAGENT  (spawned by Agent tool)                            │
  │                                                                │
  │  A fresh isolated Claude session running a specific task.    │
  │  Does NOT see parent's conversation history.                  │
  │  Returns only final text to parent.                           │
  │                                                                │
  │  Use when: you need parallel work or context isolation        │
  └────────────────────────────────────────────────────────────────┘
```

---

## Agent File Location

```
  ~/.claude/agents/<name>.md        personal agents (all projects)
  .claude/agents/<name>.md          project agents (this repo only)
```

Personal agents apply everywhere. Project agents apply in this repo.
Project agents take precedence over personal agents with the same name.

---

## Full Frontmatter Reference

```yaml
---
# ── IDENTITY ──────────────────────────────────────────────────────
name: code-reviewer
description: "Expert code reviewer: quality, security, performance, best practices"
# max 250 chars — Claude uses this to decide when to delegate here

# ── MODEL ─────────────────────────────────────────────────────────
model: sonnet
# sonnet | opus | haiku | inherit (default: inherit from session)
# Full model IDs also work: claude-sonnet-4-6

# ── PERFORMANCE ───────────────────────────────────────────────────
effort: high
# low | medium | high | max
# Controls thinking depth (Opus only, ignored for other models)

maxTurns: 20
# Stop after N agentic turns (prevent infinite loops)
# Default: unlimited

# ── TOOLS ─────────────────────────────────────────────────────────
tools: Read, Glob, Grep
# Allowlist — ONLY these tools available
# Omit to allow all tools

disallowedTools: Write, Edit, Bash
# Denylist — inherit all EXCEPT these
# Cannot use both tools: and disallowedTools simultaneously

# For agent spawning control:
# tools: Agent(researcher, reviewer)  ← only these subagents allowed
# tools: Agent                         ← any subagent allowed
# (omit Agent from tools to block all subagent spawning)

# ── PERMISSIONS ───────────────────────────────────────────────────
permissionMode: plan
# default | acceptEdits | plan | dontAsk | bypassPermissions
# Override for this agent only
# Note: bypassPermissions parent overrides this

# ── CONTEXT INJECTION ─────────────────────────────────────────────
skills: api-conventions, security-checklist
# Inject these skill file contents at startup
# Comma-separated skill names from .claude/skills/

# ── MCP SERVERS ───────────────────────────────────────────────────
mcpServers:
  playwright:
    type: stdio
    command: npx
    args: ["-y", "@playwright/mcp@latest"]
  github                      # Reference existing server by name
# Agent-scoped MCP servers — only this agent has access

# ── ISOLATION ─────────────────────────────────────────────────────
isolation: worktree
# Run agent in isolated git worktree (own branch, own directory)

# ── MEMORY ────────────────────────────────────────────────────────
memory: project
# user | project | local
# Where this agent's memory is stored

# ── EXECUTION ─────────────────────────────────────────────────────
background: false
# Run as background task (default: false)

initialPrompt: "Begin by reading ARCHITECTURE.md"
# Auto-submitted as first turn when agent starts

# ── HOOKS ─────────────────────────────────────────────────────────
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "./scripts/validate-command.sh"
  PostToolUse:
    - matcher: "Edit|Write"
      hooks:
        - type: command
          command: "npx eslint --fix"
          async: true
---

You are an expert code reviewer with 20 years of experience.

When reviewing code, ALWAYS check:
1. Security vulnerabilities (OWASP top 10)
2. Performance bottlenecks
3. Error handling completeness
4. Test coverage gaps
5. Code clarity and maintainability

Format your review as:
## Summary
## Critical Issues (must fix)
## Suggestions (nice to have)
## Verdict (approve/request changes)
```

---

## How to Invoke an Agent

### As main session (replaces your default Claude):

```bash
claude --agent code-reviewer
# Entire session runs as the code-reviewer agent
```

### As a subagent (delegated task):

Claude will automatically delegate to your agent when the task matches
its description. You can also explicitly trigger it:

```
You: "Review the auth module for security issues"
Claude: [delegates to code-reviewer agent automatically]
```

---

## Agent Discovery Order

```
  1. Agents defined in plugins (highest priority)
  2. ~/.claude/agents/*.md  (personal)
  3. .claude/agents/*.md    (project)

  If same name exists in multiple places:
  plugin > personal > project
```

---

## Designing Effective Agents

### The Single Responsibility Principle

Each agent should do ONE thing extremely well.

```
  BAD: generic-assistant.md
  description: "Helps with code, writing, analysis, and testing"
  → Claude won't know when to use it
  → Does nothing particularly well

  GOOD: security-auditor.md
  description: "Audit code for SQL injection, XSS, CSRF, auth flaws, secrets in code"
  → Clear trigger conditions
  → Deep expertise in one domain
```

### Agent Capability Matrix

```
  ┌────────────────────┬──────────┬──────────┬──────────┬──────────┐
  │  Agent Type        │  Model   │  Tools   │  Turns   │  Memory  │
  ├────────────────────┼──────────┼──────────┼──────────┼──────────┤
  │  Explorer          │  haiku   │  R,G,Gb  │  10      │  none    │
  │  (research)        │          │  (read)  │          │          │
  ├────────────────────┼──────────┼──────────┼──────────┼──────────┤
  │  Implementer       │  sonnet  │  All     │  50      │  project │
  │  (code changes)    │          │          │          │          │
  ├────────────────────┼──────────┼──────────┼──────────┼──────────┤
  │  Reviewer          │  sonnet  │  R,G,Gb  │  20      │  none    │
  │  (analysis only)   │          │  (read)  │          │          │
  ├────────────────────┼──────────┼──────────┼──────────┼──────────┤
  │  Orchestrator      │  opus    │  Agent() │  100     │  project │
  │  (coordinates)     │          │  + Read  │          │          │
  ├────────────────────┼──────────┼──────────┼──────────┼──────────┤
  │  CI Runner         │  haiku   │  Bash    │  5       │  none    │
  │  (automated)       │          │  only    │          │          │
  └────────────────────┴──────────┴──────────┴──────────┴──────────┘

  R = Read, G = Grep, Gb = Glob
```

---

## Real Agent Examples

### Security Auditor

```markdown
---
name: security-auditor
description: "Security audit: OWASP top 10, SQL injection, XSS, secrets in code, auth vulnerabilities"
model: sonnet
tools: Read, Grep, Glob, Bash
disallowedTools: Write, Edit
maxTurns: 30
permissionMode: default
---

You are a senior security engineer. Your ONLY job is finding vulnerabilities.

ALWAYS check for:
- SQL injection (look for string concatenation in queries)
- XSS (unsanitized user input in HTML)
- Secrets in code (API keys, passwords, tokens)
- Broken authentication (JWT, session, OAuth)
- Insecure direct object references
- Missing rate limiting
- Unvalidated redirects

Output format:
## CRITICAL (exploitable now)
## HIGH (needs fixing soon)
## MEDIUM (should fix)
## LOW (informational)

For each issue: file:line — description — exploit scenario — fix
```

### Fast Explorer

```markdown
---
name: explorer
description: "Quickly explore and understand codebase: find files, trace call chains, summarize modules"
model: haiku
tools: Read, Grep, Glob
maxTurns: 15
effort: low
---

You are a codebase explorer. Find and summarize information quickly.
Be concise. Return structured summaries, not prose.
```

### Database Agent

```markdown
---
name: db-agent
description: "Database operations: query optimization, migrations, schema analysis"
model: sonnet
tools: Bash, Read
mcpServers:
  postgres:
    type: stdio
    command: npx
    args: ["-y", "@anthropic-ai/mcp-server-postgres"]
    env:
      DATABASE_URL: "${DATABASE_URL}"
permissionMode: plan
maxTurns: 10
---

You are a database expert. Always:
- Explain queries before running them
- Use transactions for multi-step operations
- Verify backups exist before destructive operations
- Never run migrations without confirmation
```

### PR Reviewer

```markdown
---
name: pr-reviewer
description: "Review GitHub pull requests: code quality, test coverage, breaking changes"
model: sonnet
tools: Read, Grep, Glob, Bash
mcpServers:
  - github
maxTurns: 25
initialPrompt: "Start by reading the PR diff"
---

You are an expert PR reviewer. For every PR:

1. Read the full diff
2. Check test coverage for changed code
3. Look for breaking API changes
4. Verify error handling
5. Check for hardcoded values
6. Review naming consistency

Be constructive, specific, and provide code examples for suggested changes.
```

---

## Agent Hooks — Active Only While Agent Runs

Hooks defined in an agent's frontmatter apply ONLY when that agent is active.
When the agent finishes, those hooks are removed.

```
  Main session (no agent active):
    PreToolUse: [global hooks from settings.json]

  Agent "security-auditor" active:
    PreToolUse: [global hooks] + [agent's own hooks]

  Agent finishes, back to main session:
    PreToolUse: [global hooks only]
```

---

## Tool Restriction Patterns

```yaml
# Read-only agent (analysis, review, research)
tools: Read, Grep, Glob, WebFetch

# Write-only agent (code generation, docs)
tools: Write, Edit

# Shell-only agent (CI, testing, builds)
tools: Bash

# Full access minus dangerous ops
disallowedTools: Bash

# Orchestrator (can spawn specific subagents)
tools: Agent(explorer, implementer), Read

# Locked down agent (pure text reasoning)
tools: AskUserQuestion
```

---

## Summary

```
┌────────────────────────────────────────────────────────────────┐
│  CUSTOM AGENTS CHEATSHEET                                      │
├────────────────────────────────────────────────────────────────┤
│  Location       : .claude/agents/<name>.md                     │
│  Invoke as main : claude --agent <name>                        │
│  Auto-delegate  : matches when description fits task           │
│  Tool control   : tools: (allowlist) or disallowedTools:       │
│  Model override : model: haiku|sonnet|opus                     │
│  Turn limit     : maxTurns: N (prevent infinite loops)         │
│  Hooks          : active only while this agent runs            │
│  MCP scope      : agent-specific MCP servers                   │
│  Memory scope   : memory: user|project|local                   │
│  Isolation      : isolation: worktree for git isolation         │
└────────────────────────────────────────────────────────────────┘
```
