# 07 — Skills (Slash Commands): Frontmatter Powers

---

## What Are Skills?

Skills are reusable prompt templates that can be invoked as slash commands.
They can include instructions, shell preprocessing, argument substitution,
and even run in isolated subagents.

```
  Regular chat:
    You: "Review my code for security issues, check all TypeScript files,
          look for SQL injection, XSS, run the linter, check dependencies..."
    (You retype this every time)

  With a skill:
    You: /security-review
    (One command, always runs the same thorough checklist)
```

---

## Discovery Order

Claude Code looks for skills in this order:

```
  1  Plugin scope          plugins define skills (prefixed: plugin-name:skill)
       │
  2  Personal scope        ~/.claude/skills/*/SKILL.md
       │
  3  Project scope         .claude/skills/*/SKILL.md
       │
  4  Ancestor directories  .claude/skills/ walked up from cwd
```

---

## Skill Directory Structure

```
  .claude/skills/
  └── security-review/
      ├── SKILL.md           ← required: frontmatter + instructions
      ├── checklist.md       ← optional: supporting reference docs
      └── scripts/
          └── run-audit.sh   ← optional: helper scripts
```

---

## SKILL.md — Full Frontmatter Reference

```yaml
---
# ── IDENTITY ───────────────────────────────────────────────────
name: security-review
description: "Review code for OWASP top 10 vulnerabilities, run linter, check deps"
# max 250 chars — used to decide when to auto-invoke this skill

argument-hint: "[file-pattern] [--fix]"
# shown in autocomplete when user types /security-review

# ── INVOCATION CONTROL ─────────────────────────────────────────
user-invocable: true
# user can call /security-review (default: true)

disable-model-invocation: true
# Claude will NOT auto-invoke this skill — only user can trigger it

# ── EXECUTION CONTEXT ──────────────────────────────────────────
context: fork
# run in isolated subagent — won't pollute main context window

agent: Explore
# which subagent type: Explore | Plan | general-purpose

model: opus
# model override: sonnet | opus | haiku | inherit

effort: high
# effort level: low | medium | high | max

# ── ACCESS CONTROL ─────────────────────────────────────────────
allowed-tools: Read, Grep, Bash
# restrict which tools this skill can use
# omit to allow all tools

# ── AUTO-LOADING ───────────────────────────────────────────────
paths: "src/**/*.ts"
# only make this skill available when Claude reads matching files

# ── SHELL ──────────────────────────────────────────────────────
shell: bash
# shell to use for !`...` preprocessing: bash | powershell
---

Security review instructions go here...
```

---

## String Substitutions

Inside skill content, these placeholders are replaced at invocation time:

```
  $ARGUMENTS              all text after /skill-name
  $ARGUMENTS[0] or $1     first argument
  $ARGUMENTS[1] or $2     second argument
  $ARGUMENTS[2] or $3     third argument
  ${CLAUDE_SESSION_ID}    current session's unique ID
  ${CLAUDE_SKILL_DIR}     absolute path to this skill's directory
```

Example:
```markdown
---
name: explain
argument-hint: "<function-name> [--detail high|low]"
---

Explain the function `$1` in detail level: $2

Read the source file and provide:
- What it does
- Parameters and return type
- Example usage
- Edge cases
```

Invocation: `/explain parseJWT --detail high`
Result: `$1` → `parseJWT`, `$2` → `--detail high`

---

## Shell Preprocessing — The Hidden Power

Lines starting with `` !` `` run shell commands BEFORE the skill content
is sent to Claude. The output replaces the placeholder inline.

```markdown
---
name: git-review
---

Review the following changes for quality and correctness:

Recent git log:
!`git log --oneline -10`

Files changed in this PR:
!`git diff --name-only origin/main...HEAD`

Full diff:
!`git diff origin/main...HEAD`
```

What Claude receives (after preprocessing):
```
Review the following changes for quality and correctness:

Recent git log:
a3f9c12 fix: correct auth token expiry
b2e1d45 feat: add refresh token support
...

Files changed in this PR:
src/auth/token.ts
src/auth/refresh.ts
tests/auth.test.ts

Full diff:
--- a/src/auth/token.ts
+++ b/src/auth/token.ts
...
```

This runs BEFORE Claude reads it — not as a tool call during the session.

---

## Context Fork — Isolated Subagent

Without `context: fork`:
```
  Main session context window:
  ┌──────────────────────────────────────────────────────────┐
  │  Your conversation...                                    │
  │  Skill content...                                        │
  │  All tool results from skill...   ← consumes your window │
  │  More conversation...                                    │
  └──────────────────────────────────────────────────────────┘
```

With `context: fork`:
```
  Main session:                    Subagent (isolated):
  ┌────────────────────┐           ┌────────────────────────┐
  │  Your conversation │           │  Skill content         │
  │                    │  spawns ──►  Tool calls            │
  │  [subagent runs]   │           │  File reads            │
  │                    │◄──result──│  Analysis              │
  │  Summary returned  │           │  All context used here │
  └────────────────────┘           └────────────────────────┘
```

Use `context: fork` for:
- Skills that read many files (codebase analysis)
- Long-running research tasks
- Security reviews
- Anything that would bloat main context

---

## Auto-Invocation vs Manual Invocation

```
  MANUAL (user-invocable: true — default):
    /security-review
    /explain parseJWT
    @skill-name in your message

  AUTO (Claude decides when to invoke):
    Claude reads the skill description during session
    If description matches current task, Claude uses the skill
    Controlled by: description quality (max 250 chars)

  DISABLE auto-invocation:
    disable-model-invocation: true
    → Only user can trigger, Claude never auto-invokes
```

### Description Quality Matters

```
  BAD description (too vague):
    "Review code"
    → Claude won't know when to auto-invoke

  GOOD description (specific trigger):
    "Review TypeScript/Python code for OWASP top 10 security issues,
    SQL injection, XSS, run ESLint, and check npm audit"
    → Claude knows exactly when to auto-invoke
```

---

## Path-Scoped Skills

A skill with `paths:` frontmatter only appears when Claude is working
with files matching those patterns.

```yaml
---
name: react-component-review
paths: "src/components/**/*.tsx"
description: "Review React component for hooks rules, prop types, accessibility"
---
```

```
  Claude reads: src/utils/helpers.ts
  → react-component-review skill NOT available

  Claude reads: src/components/Button.tsx
  → react-component-review skill IS available
```

---

## Practical Skill Examples

### Example 1 — Daily Standup

```markdown
---
name: standup
description: "Generate standup update from recent git commits and open tasks"
context: fork
---

Generate a standup update for today.

Recent commits:
!`git log --oneline --since="24 hours ago" --author="$(git config user.email)"`

Open PRs:
!`gh pr list --author "@me" --state open`

Format as:
- **Yesterday:** [what was done based on commits]
- **Today:** [what's planned based on open PRs]
- **Blockers:** [any if visible]
```

### Example 2 — Test a Specific Function

```markdown
---
name: test-fn
argument-hint: "<function-name>"
allowed-tools: Read, Grep, Bash
---

Write comprehensive tests for the function `$1`.

1. Find the function definition with Grep
2. Read the file to understand its behavior
3. Write unit tests covering:
   - Happy path
   - Edge cases
   - Error conditions
4. Run the tests to confirm they pass
```

### Example 3 — Deploy Checklist (manual only)

```markdown
---
name: pre-deploy
disable-model-invocation: true
description: "Pre-deployment checklist: tests, build, migrations check"
allowed-tools: Bash
---

Run the pre-deployment checklist:

1. Run all tests: `npm test`
2. Build the project: `npm run build`
3. Check for pending migrations: `npm run db:status`
4. Check for TODO/FIXME in changed files:
   !`git diff --name-only origin/main...HEAD | xargs grep -l "TODO\|FIXME" 2>/dev/null || echo "none"`
5. Report results and flag any failures
```

---

## Context Budget

```
  Skills descriptions are always in context (so Claude knows when to invoke them)
  Full skill content loads only when invoked

  Budget for descriptions:
    ~1% of total context window
    Minimum: 8KB
    Per-skill cap: 250 characters

  If you have 50 skills × 250 chars = 12,500 chars
  Still well within budget
```

---

## Summary

```
┌────────────────────────────────────────────────────────────────┐
│  SKILLS CHEATSHEET                                             │
├────────────────────────────────────────────────────────────────┤
│  Invoke            : /skill-name [args] or @skill-name         │
│  Discovery         : plugin > ~/.claude/skills > .claude/skills│
│  Shell preprocess  : !`command` in content (runs before Claude)│
│  Arg substitution  : $1, $2, $ARGUMENTS, ${CLAUDE_SKILL_DIR}   │
│  Isolation         : context: fork (runs in subagent)          │
│  Path-scoping      : paths: "src/**/*.ts" frontmatter          │
│  No auto-invoke    : disable-model-invocation: true            │
│  Tool restriction  : allowed-tools: Read, Grep                 │
│  Description limit : 250 chars max (auto-invoke accuracy)      │
└────────────────────────────────────────────────────────────────┘
```
