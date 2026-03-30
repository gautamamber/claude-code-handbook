# 08 — Worktrees: True Parallelism

---

## The Problem Without Worktrees

One Claude session, one copy of the repo. You can't work on two things at once.

```
  Without worktrees:

  You:   "Fix the auth bug"
  Claude: [edits src/auth/token.ts]
          [edits tests/auth.test.ts]
          Working...

  Meanwhile:
  You:   "Also, can you add the new dashboard feature?"
  Claude: [has to wait — same files, same branch, conflicts]

  Result: Sequential work only. One thing at a time.
```

---

## What Worktrees Enable

```
  With worktrees:

  Session A (main):                  Session B (worktree):
  branch: main                       branch: fix/auth-bug
  dir: /Users/alice/work/myapp/      dir: /tmp/worktrees/myapp-abc123/

  Claude fixes auth bug ──────────►  Claude builds dashboard feature
  (independent files)                (independent files)

  Both run SIMULTANEOUSLY. No conflicts.
```

---

## What Is a Git Worktree?

A git worktree is an additional working directory attached to the same repository.
Each worktree has its own branch and its own set of files on disk.

```
  Your git repo:  /Users/alice/work/myapp/
                  branch: main

  Worktrees:
    /tmp/worktrees/myapp-abc123/    ← branch: fix/auth-bug
    /tmp/worktrees/myapp-def456/    ← branch: feature/dashboard

  Same .git directory. Different working files. Different branches.
  Changes in one worktree don't affect the others.
```

---

## How to Use Worktrees with Claude Code

### Option 1 — CLI Flag

```bash
# Start a new isolated Claude session in a worktree
claude --worktree
```

Claude Code automatically:
1. Creates a new git branch (named from task description)
2. Creates a worktree at a temp path
3. Opens the session in that worktree
4. Cleans up when session ends (if no changes)

### Option 2 — Subagent Isolation

In `.claude/agents/` or when using the Agent tool:

```json
{
  "isolation": "worktree"
}
```

Or in agent frontmatter:
```yaml
---
isolation: worktree
---
```

---

## Worktree Lifecycle

```
  User: "Fix the auth bug in parallel with my current work"
                │
                ▼
        WorktreeCreate hook fires
                │
                ▼
  ┌─────────────────────────────────────────┐
  │  New worktree created:                  │
  │  path: /tmp/worktrees/myapp-a1b2c3/     │
  │  branch: claude/fix-auth-bug            │
  └─────────────────────────────────────────┘
                │
                ▼
  Subagent works in isolation:
    - reads files from worktree path
    - edits files in worktree
    - runs tests in worktree
    - cannot affect main session files
                │
                ▼
         Subagent finishes
                │
           ┌────┴────┐
       No changes    Changes made
           │              │
           ▼              ▼
  Worktree deleted   Worktree preserved
  automatically      Branch: claude/fix-auth-bug
                     Path returned to parent session
                           │
                           ▼
                WorktreeRemove hook fires
                           │
                           ▼
              User reviews branch and merges
```

---

## Shared Memory Across Worktrees

```
  /Users/alice/work/myapp/          (main checkout)
  /tmp/worktrees/myapp-abc123/      (worktree A)
  /tmp/worktrees/myapp-def456/      (worktree B)

  All share the same memory directory:
  ~/.claude/projects/<myapp-id>/memory/

  Why? All have the same git repo root.
  Memory is scoped to the repo, not the checkout path.

  This means:
  ✓ Feedback written in worktree A is available in worktree B
  ✓ All sessions on the same project benefit from the same memories
```

---

## Hook Events for Worktrees

```
  WorktreeCreate
    fires when: a new worktree is about to be created
    input includes: proposed branch name, base branch, path

  WorktreeRemove
    fires when: a worktree is about to be removed
    input includes: worktree path, branch name, had_changes flag
```

You can use these hooks to:
- Log worktree activity
- Push branches to remote on creation
- Notify team via Slack when a parallel task starts
- Clean up resources when worktrees are removed

---

## Subagent Transcripts — Separate Storage

When a worktree session uses a subagent, the conversation is stored separately:

```
  ~/.claude/projects/{project}/
  └── {sessionId}/
      └── subagents/
          └── agent-{agentId}.jsonl   ← subagent transcript

  Main session transcript:
  ~/.claude/projects/{project}/{sessionId}/transcript.jsonl
```

Important: Context compaction in the main session does NOT affect subagent transcripts.
They persist independently.

---

## Real-World Parallel Workflow

```
  Main terminal:                     Background terminal:
  claude (main session)              claude --worktree

  ┌────────────────────┐             ┌──────────────────────────┐
  │ You're writing     │             │ Claude is:               │
  │ new features       │             │ • running full test suite│
  │                    │             │ • fixing lint errors     │
  │ branch: main       │             │ • updating dependencies  │
  └────────────────────┘             │ branch: claude/ci-fixes  │
                                     └──────────────────────────┘

  Both complete → you review the worktree branch → merge
```

---

## When NOT to Use Worktrees

```
  ✗ Simple single-file edits (overkill)
  ✗ When you need the changes to be immediately visible in main session
  ✗ When the task requires the main session's current context/state
  ✗ Very short tasks (overhead isn't worth it)

  ✓ Long-running independent tasks
  ✓ Running tests while you continue working
  ✓ Large refactors that touch many files
  ✓ Experimental changes you might throw away
  ✓ CI/CD-style automated checks
```

---

## Summary

```
┌────────────────────────────────────────────────────────────────┐
│  WORKTREES CHEATSHEET                                          │
├────────────────────────────────────────────────────────────────┤
│  CLI flag           : claude --worktree                        │
│  Agent isolation    : isolation: "worktree" in frontmatter     │
│  Auto-cleanup       : yes, if no changes made                  │
│  Branch naming      : claude/<task-description>               │
│  Shared memory      : yes — all worktrees share repo memory    │
│  Hooks              : WorktreeCreate | WorktreeRemove          │
│  Transcripts        : stored separately, survive compaction    │
│  Best use case      : parallel independent long-running tasks  │
└────────────────────────────────────────────────────────────────┘
```
