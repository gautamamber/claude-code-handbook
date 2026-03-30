# 06 — Memory System: Machine-Local, Project-Scoped

---

## What Is the Memory System?

Claude starts every session with zero memory of past sessions.
The auto memory system is a file-based workaround:
Claude writes notes to disk that get loaded at the start of future sessions.

```
  Session 1:
    User: "We always use snake_case for Python vars"
    Claude: [writes to memory file]

  Session 2 (new conversation):
    MEMORY.md loaded at startup
    Claude already knows: snake_case for Python vars
    No need to repeat it
```

---

## Directory Structure

```
  ~/.claude/
  └── projects/
      └── <project-id>/
          └── memory/
              ├── MEMORY.md              ← INDEX (loaded at every session start)
              ├── user_role.md           ← topic file (loaded on demand)
              ├── feedback_testing.md    ← topic file (loaded on demand)
              ├── project_context.md     ← topic file (loaded on demand)
              └── api_conventions.md     ← topic file (loaded on demand)
```

---

## The Two-Tier Loading Model

Most users think everything in memory loads at startup. That's wrong.

```
  SESSION START
       │
       ▼
  ┌──────────────────────────────────────────────────────┐
  │  TIER 1: Always Loaded                               │
  │                                                      │
  │  MEMORY.md  (first 200 lines OR 25KB)               │
  │                                                      │
  │  This is the INDEX. It contains:                     │
  │  • Links to topic files                              │
  │  • One-line descriptions of each memory             │
  │  • Must stay concise or entries get cut off          │
  └──────────────────────────────────────────────────────┘
       │
       │  Later in session...
       ▼
  ┌──────────────────────────────────────────────────────┐
  │  TIER 2: Loaded On Demand                            │
  │                                                      │
  │  user_role.md, feedback_testing.md, etc.             │
  │                                                      │
  │  Loaded only when Claude explicitly reads them.      │
  │  Not in context at session start.                    │
  │  Claude reads them when the topic becomes relevant.  │
  └──────────────────────────────────────────────────────┘
```

---

## How Project ID Is Derived

```
  ┌─────────────────────────────────────────────────────────────┐
  │  Git repository?                                            │
  │                                                             │
  │  YES → ID = derived from git repo root                      │
  │                                                             │
  │  All of these share ONE memory directory:                   │
  │    /Users/alice/work/myapp/           ← repo root          │
  │    /Users/alice/work/myapp/src/       ← subdirectory        │
  │    /Users/alice/work/myapp/tests/     ← subdirectory        │
  │    /tmp/worktrees/myapp-feature-x/    ← worktree            │
  │                                                             │
  │  NO  → ID = absolute path of project root                  │
  └─────────────────────────────────────────────────────────────┘
```

---

## Memory Types

```
  ┌────────────────┬──────────────────────────────────────────────┐
  │  Type          │  What to Store                               │
  ├────────────────┼──────────────────────────────────────────────┤
  │  user          │  Role, goals, expertise, preferences         │
  │                │  "User is a senior Go dev, new to React"     │
  ├────────────────┼──────────────────────────────────────────────┤
  │  feedback      │  Corrections Claude must not repeat          │
  │                │  "Don't mock DB in tests — we got burned"    │
  ├────────────────┼──────────────────────────────────────────────┤
  │  project       │  Ongoing work, decisions, deadlines          │
  │                │  "Auth rewrite due 2026-04-01, legal req"     │
  ├────────────────┼──────────────────────────────────────────────┤
  │  reference     │  Pointers to external systems                │
  │                │  "Bugs tracked in Linear project INGEST"     │
  └────────────────┴──────────────────────────────────────────────┘
```

---

## Memory File Format

Every memory is its own file with YAML frontmatter:

```markdown
---
name: feedback_no_db_mocks
description: Never mock the database in integration tests — use real DB always
type: feedback
---

Don't mock the database in integration tests.

**Why:** Got burned in Q3 2025 when mocked tests passed but a prod migration
failed — the mock didn't capture the schema difference.

**How to apply:** When writing or reviewing integration tests, always use a real
test database (test container or local instance). Only mock external APIs and
email services, not the DB.
```

---

## MEMORY.md — The Index File

MEMORY.md is the file that loads at session start. It's an index, not content.

```markdown
# Memory Index

## User
- [user_role.md](user_role.md) — Senior Go dev, new to React, prefers concise explanations

## Feedback
- [feedback_no_db_mocks.md](feedback_no_db_mocks.md) — Never mock DB in integration tests
- [feedback_no_summary.md](feedback_no_summary.md) — Don't summarize at end of responses
- [feedback_snake_case.md](feedback_snake_case.md) — Python: always snake_case, never camelCase

## Project
- [project_auth_rewrite.md](project_auth_rewrite.md) — Auth rewrite: legal req, due 2026-04-01

## References
- [ref_linear.md](ref_linear.md) — Pipeline bugs tracked in Linear project "INGEST"
- [ref_grafana.md](ref_grafana.md) — Oncall latency dashboard: grafana.internal/d/api-latency
```

**Critical constraint:** MEMORY.md only loads first 200 lines / 25KB.
If it grows beyond that, entries at the bottom are silently ignored.
Keep it as a lean index. Put all content in separate topic files.

---

## What Claude Writes vs What It Skips

```
  WRITE to memory:
    ✓ User corrects Claude ("don't do X, do Y instead")
    ✓ User shares their role, expertise, preferences
    ✓ Project decisions with long-term impact
    ✓ Pointers to external systems (Linear project, Grafana URL)
    ✓ Recurring patterns Claude gets wrong

  DO NOT write to memory:
    ✗ Code patterns (derivable from codebase)
    ✗ Git history (use git log)
    ✗ Current session tasks (use task system)
    ✗ Things already in CLAUDE.md
    ✗ Temporary state ("working on X right now")
    ✗ Debugging solutions (commit message has this)
```

---

## Memory Is Machine-Local

```
  Machine A  ──►  ~/.claude/projects/myapp/memory/   ← has memories
      │
      │  git push / git pull
      ▼
  Machine B  ──►  ~/.claude/projects/myapp/memory/   ← EMPTY
                  (memory not in git, not synced)
```

Memory does not sync across machines, cloud, or team members.
Each developer has their own memory for each project.

---

## Configuration

```json
{
  "autoMemoryEnabled": true,
  "autoMemoryDirectory": "~/custom-memory-location"
}
```

Disable memory entirely:
```json
{
  "autoMemoryEnabled": false
}
```

---

## Memory Lifecycle in a Session

```
  Session Start
       │
       ├──► Load MEMORY.md (first 200 lines)
       │    Claude has access to the index
       │
       │  During session...
       │
       ├──► User corrects Claude
       │    Claude writes feedback_*.md
       │    Claude updates MEMORY.md index
       │
       ├──► User mentions their role
       │    Claude writes/updates user_role.md
       │    Claude updates MEMORY.md index
       │
       └──► Session ends
            Memory persists to next session
```

---

## How to Ask Claude to Remember Something

```
  You: "Remember: we use Podman not Docker on this project"
  Claude: [writes to memory immediately]

  You: "Forget that thing about snake_case"
  Claude: [finds and removes relevant memory entry]

  You: "What do you remember about me?"
  Claude: [reads MEMORY.md + loads relevant topic files]
```

---

## Feedback Memory — Most Impactful Type

Feedback memories prevent Claude from repeating mistakes you've already corrected.

```
  WITHOUT feedback memory:

  Session 1: You: "Don't add trailing summaries"  Claude: fixes it
  Session 2: Claude adds trailing summary again
  Session 3: You: "I said don't add trailing summaries!"
  Session 4: Claude adds trailing summary again...

  WITH feedback memory:

  Session 1: You: "Don't add trailing summaries"
             Claude: writes feedback_no_summary.md
  Session 2+: MEMORY.md loaded → feedback_no_summary.md read
              Claude never adds trailing summaries again
```

---

## Summary

```
┌────────────────────────────────────────────────────────────────┐
│  MEMORY CHEATSHEET                                             │
├────────────────────────────────────────────────────────────────┤
│  Location       : ~/.claude/projects/<repo-id>/memory/         │
│  Index file     : MEMORY.md (loaded at startup)               │
│  Index limit    : 200 lines OR 25 KB                           │
│  Topic files    : loaded on demand, not at startup            │
│  Project scope  : one dir per git repo (shared by worktrees)   │
│  Sync           : machine-local ONLY, not synced               │
│  Types          : user | feedback | project | reference        │
│  Most impactful : feedback type (prevents repeated mistakes)   │
│  Disable        : "autoMemoryEnabled": false in settings       │
└────────────────────────────────────────────────────────────────┘
```
