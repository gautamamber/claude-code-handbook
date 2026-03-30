# 04 — Permission Modes: How Evaluation Actually Works

---

## Overview

Every time Claude wants to use a tool, a permission check happens.
Understanding the exact evaluation order — and what each mode does — is essential.

---

## The 6 Permission Modes

```
┌─────────────────────────────────────────────────────────────────────┐
│  MODE: default                                                      │
│                                                                     │
│  Asks before file edits and shell commands.                         │
│  Read-only operations (Read, Glob, Grep) are auto-allowed.         │
│                                                                     │
│  [Read file] → auto-allowed                                         │
│  [Edit file] → PROMPT: "Allow Claude to edit this file?"           │
│  [Bash cmd]  → PROMPT: "Allow Claude to run this command?"         │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│  MODE: acceptEdits                                                  │
│                                                                     │
│  Auto-accepts all file edits. Still prompts for shell commands.     │
│                                                                     │
│  [Read file] → auto-allowed                                         │
│  [Edit file] → auto-allowed (no prompt)                             │
│  [Bash cmd]  → PROMPT                                               │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│  MODE: plan                                                         │
│                                                                     │
│  Read-only. Claude can research but not change anything.            │
│  Creates PLAN.md with proposed approach. User approves to proceed. │
│                                                                     │
│  [Read file] → auto-allowed                                         │
│  [Edit file] → BLOCKED (plan mode only reads)                      │
│  [Bash cmd]  → BLOCKED                                              │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│  MODE: auto                                                         │
│                                                                     │
│  Background classifier evaluates each action.                       │
│  Safe actions auto-allowed. Risky actions auto-denied or prompted. │
│  Requires Team+ plan. Uses Claude Sonnet 4.6 as classifier.        │
│                                                                     │
│  [Read file] → auto-allowed                                         │
│  [Edit file] → classifier decides                                   │
│  [Bash cmd]  → classifier decides                                   │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│  MODE: dontAsk                                                      │
│                                                                     │
│  Auto-DENIES everything not explicitly in allow rules.              │
│  Useful for read-only audits or strict controlled environments.    │
│                                                                     │
│  [Any tool]  → DENIED unless in allow list                         │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│  MODE: bypassPermissions                                            │
│                                                                     │
│  Skips ALL permission checks. Dangerous.                            │
│  Only use inside containers or VMs. Not for local development.     │
│                                                                     │
│  [Any tool]  → auto-allowed (no checks at all)                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Evaluation Order — Every Tool Call

This is the exact decision tree Claude Code runs for each tool use:

```
  Claude wants to use a tool
            │
            ▼
  ┌─────────────────────────┐
  │  Check explicit ALLOW   │  ← settings.json permissions.allow
  │  rules                  │
  └──────────┬──────────────┘
             │
         matches?
        /        \
      YES          NO
       │            │
       ▼            ▼
    ALLOW     ┌─────────────────────────┐
              │  Check explicit DENY    │  ← settings.json permissions.deny
              │  rules                  │
              └──────────┬──────────────┘
                         │
                     matches?
                    /        \
                  YES          NO
                   │            │
                   ▼            ▼
                DENY      ┌─────────────────────────┐
                          │  Check explicit ASK     │  ← settings.json permissions.ask
                          │  rules                  │
                          └──────────┬──────────────┘
                                     │
                                 matches?
                                /        \
                              YES          NO
                               │            │
                               ▼            ▼
                          PROMPT USER   Apply mode default
                                              │
                              ┌───────────────┼──────────────────┐
                              │               │                  │
                           default       acceptEdits           auto
                              │               │                  │
                           prompt         allow file        classifier
                                           edits            evaluates
```

---

## Auto Mode — The Background Classifier

Auto mode uses a second Claude instance to evaluate each action.

```
Your session model: claude-opus-4-6

          Claude wants to: git push origin main
                  │
                  ▼
        ┌──────────────────────┐
        │  Background call to  │
        │  Claude Sonnet 4.6   │  ← always Sonnet 4.6, regardless of your model
        │                      │
        │  Input:              │
        │  - your messages     │
        │  - pending action    │
        │                      │
        │  NOTE: tool results  │
        │  and file contents   │
        │  are STRIPPED from   │
        │  input (prevents     │
        │  prompt injection)   │
        └──────────┬───────────┘
                   │
         classifier decides
        /                    \
     ALLOW                  DENY
       │                      │
   Action runs          Action blocked
                              │
                    Claude gets feedback,
                    tries another approach
```

### Auto Mode Fallback

```
If classifier blocks 3 times in a row    → switch back to prompting
If classifier blocks 20 times in session → switch back to prompting
```

### What the Classifier Blocks by Default

```
  curl <url> | bash                  downloading and executing remote code
  pip install <unknown>              installing from untrusted sources
  POST to external APIs              sending sensitive data outward
  git push origin main               pushing to main/master
  git push --force                   force push anywhere
  rm on production paths             mass deletion
  AWS/GCP/IAM commands               cloud infrastructure changes
  Production deploy commands          deployments
  Database migration commands         data migrations
```

### What the Classifier Allows by Default

```
  Local file read/write/edit          in working directory
  npm/pip install from lock files     locked dependencies only
  git commit, git branch, git status  local git operations
  Read-only HTTP requests (GET)       fetching data
  git push to current branch          non-main branches
  git push to Claude-created branches any branch Claude made
```

---

## What Auto Mode Drops from Your Allow Rules

When you enter auto mode, these allow rules are temporarily suspended:

```
  Bash(*)              ← blanket shell access removed
  Bash(python *)       ← interpreter access removed
  Bash(node *)         ← interpreter access removed
  Package manager run commands (e.g., npm run *)
  Agent(*)             ← all agent spawning removed
```

These are restored when you exit auto mode.

---

## Plan Mode — How It Works

```
  User: "Refactor the auth module"
            │
            ▼
  ┌─────────────────────┐
  │   plan mode active  │
  │   (read-only)       │
  └────────┬────────────┘
           │
           ▼
  Claude researches codebase
  (Read, Glob, Grep only)
           │
           ▼
  ┌─────────────────────┐
  │   Creates PLAN.md   │
  │                     │
  │   ## Approach       │
  │   1. Extract...     │
  │   2. Move...        │
  │   3. Update tests   │
  └────────┬────────────┘
           │
           ▼
  User reviews PLAN.md
           │
      ┌────┴────┐
      │         │
   Approve    Refine
      │
      ▼
  User picks execution mode:
  • Approve + auto mode
  • Approve + acceptEdits
  • Approve + default (manual)
```

---

## Permissions vs Sandbox

These are two different enforcement layers. Both apply independently.

```
  ┌──────────────────────────────────────────────────────────────┐
  │  PERMISSIONS LAYER                                           │
  │                                                              │
  │  Controls what Claude CHOOSES to do.                         │
  │  Evaluated before tool execution.                            │
  │  Can be configured via allow/deny/ask rules.                │
  │                                                              │
  │  Example: deny rule for "rm -rf"                            │
  │  → Claude is told it cannot run this                        │
  └──────────────────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────────────┐
  │  SANDBOX LAYER                                               │
  │                                                              │
  │  Controls what the Bash tool CAN DO at OS level.            │
  │  Filesystem isolation: only working dir accessible.          │
  │  Network isolation: only allowlisted domains.               │
  │                                                              │
  │  Example: sandbox enabled                                   │
  │  → rm -rf outside working dir fails at OS level             │
  │  → Even if Claude tries, the OS blocks it                   │
  └──────────────────────────────────────────────────────────────┘

  deny rules    → Claude won't try
  sandbox       → even if Claude tries, it physically can't
  Best setup    → both layers together
```

---

## Subagent Permission Inheritance

When Claude spawns a subagent, permissions flow like this:

```
  Parent session (mode: auto)
            │
            │ spawns
            ▼
  Subagent (inherits auto)
            │
            ├── can override with permissionMode in frontmatter
            │   EXCEPT: cannot override if parent is bypassPermissions
            │
            └── classifier checks subagent actions SEPARATELY
                from parent session checks
```

---

## Setting Permission Mode

```json
// In settings.json:
{
  "permissions": {
    "defaultMode": "auto"
  }
}

// Or in CLI:
// claude --permission-mode auto

// Or interactively:
// /permission-mode auto
```

---

## Deny Takes Precedence Over Hooks

```
Scenario:
  - Hook says: permissionDecision = "allow"
  - Deny rule matches the same action

Result:
  DENY wins. Hook approval is overridden.

Order of precedence:
  deny rules > hook approvals > allow rules > mode default
```

---

## Summary

```
┌────────────────────────────────────────────────────────────────┐
│  PERMISSIONS CHEATSHEET                                        │
├────────────────────────────────────────────────────────────────┤
│  Evaluation order : allow → deny → ask → mode default          │
│  Auto mode model  : always Claude Sonnet 4.6                   │
│  Auto fallback    : 3 consecutive OR 20 total blocks           │
│  Deny vs hooks    : deny rules always win                      │
│  Sandbox          : OS-level enforcement (separate layer)       │
│  Plan mode        : read-only + creates PLAN.md                │
│  bypassPermissions: only in containers/VMs, never local        │
│  Auto drops       : Bash(*), Bash(python*), Agent allow rules  │
└────────────────────────────────────────────────────────────────┘
```
