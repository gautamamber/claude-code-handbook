# 10 — Key File Paths Reference

---

## Complete File System Map

```
  ~/ (home directory)
  └── .claude/
      ├── settings.json            ← user global settings
      ├── CLAUDE.md                ← user global instructions
      ├── keybindings.json         ← global key bindings
      ├── .mcp.json                ← user MCP server definitions
      │
      ├── skills/                  ← personal skills (slash commands)
      │   └── my-skill/
      │       └── SKILL.md
      │
      ├── agents/                  ← personal subagent definitions
      │   └── my-agent.md
      │
      └── projects/               ← per-project memory & transcripts
          └── <project-id>/
              ├── memory/
              │   ├── MEMORY.md        ← memory index
              │   ├── user_role.md     ← memory topic files
              │   └── feedback_*.md
              │
              └── <session-id>/
                  ├── transcript.jsonl  ← session transcript
                  └── subagents/
                      └── agent-<id>.jsonl
```

```
  [project root]/
  ├── CLAUDE.md                    ← project instructions (alternative location)
  ├── .mcp.json                    ← project MCP servers (commit to git)
  │
  └── .claude/
      ├── settings.json            ← shared project settings (commit to git)
      ├── settings.local.json      ← local-only settings (gitignore this)
      ├── CLAUDE.md                ← project instructions (preferred location)
      ├── keybindings.local.json   ← local key bindings
      │
      ├── rules/                   ← path-scoped instruction files
      │   ├── typescript.md
      │   ├── python.md
      │   └── tests.md
      │
      ├── skills/                  ← project slash commands
      │   └── security-review/
      │       └── SKILL.md
      │
      └── agents/                  ← project subagent definitions
          └── code-reviewer.md
```

---

## File Purpose Quick Reference

```
  FILE                              PURPOSE
  ──────────────────────────────────────────────────────────────────
  ~/.claude/settings.json           Global user settings (all projects)
  ~/.claude/CLAUDE.md               Personal instructions (all projects)
  ~/.claude/keybindings.json        Personal key bindings
  ~/.claude/.mcp.json               Personal MCP servers (all projects)

  .claude/settings.json             Project settings — COMMIT TO GIT
  .claude/settings.local.json       Local project settings — GITIGNORE
  .claude/CLAUDE.md                 Project instructions — COMMIT TO GIT
  .mcp.json                         Project MCP servers — COMMIT TO GIT

  .claude/rules/*.md                Path-scoped rules — COMMIT TO GIT
  .claude/skills/*/SKILL.md         Project skills — COMMIT TO GIT
  .claude/agents/*.md               Project subagents — COMMIT TO GIT

  ~/.claude/projects/<id>/memory/   Auto memory — NEVER COMMITTED
```

---

## Managed Policy Paths (Enterprise / IT)

```
  macOS:
    /Library/Application Support/ClaudeCode/CLAUDE.md
    /Library/Application Support/ClaudeCode/managed-settings.json

  Linux:
    /etc/claude-code/CLAUDE.md
    /etc/claude-code/managed-settings.json

  Windows:
    C:\Program Files\ClaudeCode\CLAUDE.md
    C:\Program Files\ClaudeCode\managed-settings.json
```

These files:
- Are set by IT administrators
- Override all user and project settings
- Cannot be excluded via `claudeMdExcludes`
- Cannot be overridden by any lower scope

---

## Which Files to Commit vs Gitignore

```
  ✓ COMMIT THESE (shared with team):
    .claude/settings.json
    .claude/CLAUDE.md
    .claude/rules/*.md
    .claude/skills/**
    .claude/agents/**
    .mcp.json

  ✗ GITIGNORE THESE (machine-specific):
    .claude/settings.local.json
    .claude/keybindings.local.json

  ✗ NEVER IN GIT (auto-created, machine-local):
    ~/.claude/projects/<id>/memory/
    ~/.claude/projects/<id>/<session>/transcript.jsonl
```

### Recommended .gitignore Entry

```gitignore
# Claude Code local/personal settings
.claude/settings.local.json
.claude/keybindings.local.json
```

---

## Memory Location Details

Memory is stored at:

```
  ~/.claude/projects/<project-id>/memory/
```

Where `<project-id>` is:
- **Git repo:** derived from git repository root path
- **No git:** derived from absolute project root path

All of these map to the SAME memory directory (same git repo):
```
  /Users/alice/work/myapp/           → ~/.claude/projects/abc123/memory/
  /Users/alice/work/myapp/src/       → ~/.claude/projects/abc123/memory/
  /Users/alice/work/myapp/tests/     → ~/.claude/projects/abc123/memory/
  /tmp/worktrees/myapp-xyz/          → ~/.claude/projects/abc123/memory/
```

---

## Transcript Storage

Each session's full conversation is stored as a JSONL file:

```
  ~/.claude/projects/<project-id>/<session-id>/transcript.jsonl
```

Subagent transcripts:
```
  ~/.claude/projects/<project-id>/<session-id>/subagents/agent-<id>.jsonl
```

These are:
- Written incrementally during the session
- Persistent after session ends
- NOT affected by context compaction
- Machine-local only

---

## Settings Precedence Visualized

```
  ~/.claude/settings.json         (priority 4 — lowest)
       ↑ overridden by
  .claude/settings.json           (priority 3)
       ↑ overridden by
  .claude/settings.local.json     (priority 2)
       ↑ overridden by
  managed-settings.json           (priority 1 — highest, cannot override)
```

For arrays (allow/deny/hooks): all scopes merge together.
For primitives (defaultModel, etc.): highest priority wins.

---

## CLAUDE.md Precedence Visualized

```
  ~/.claude/CLAUDE.md             (loaded — user scope)
  +
  ./.claude/CLAUDE.md             (loaded — project scope)
  +
  ./src/CLAUDE.md                 (loaded lazily — when reading src/ files)
  +
  managed policy CLAUDE.md        (always loaded, highest priority)
  =
  All injected as one user message, priority order: managed > project > user
```

---

## MCP Config Precedence

```
  ~/.claude/.mcp.json             (user scope — all projects)
  +
  .mcp.json                       (project scope — this project)
  =
  Merged: project servers override user servers with same key
```

---

## Skill Discovery Order (Full Path)

```
  1. Plugin skills:
     [plugin-install-dir]/skills/*/SKILL.md

  2. Personal skills:
     ~/.claude/skills/*/SKILL.md

  3. Project skills:
     .claude/skills/*/SKILL.md
     (also searched in ancestor directories up to repo root)
```

---

## Quick Cheatsheet: What Lives Where

```
  WANT TO...                        FILE TO EDIT
  ──────────────────────────────────────────────────────────────
  Personal coding style rules       ~/.claude/CLAUDE.md
  Team coding standards             .claude/CLAUDE.md
  Path-specific rules (TS only)     .claude/rules/typescript.md
  Allow a shell command always       ~/.claude/settings.json (allow)
  Allow a command for this project  .claude/settings.json (allow)
  Block a dangerous command         .claude/settings.json (deny)
  Add GitHub MCP server             .mcp.json
  Add personal MCP server           ~/.claude/.mcp.json
  Create a slash command            .claude/skills/name/SKILL.md
  Personal slash command            ~/.claude/skills/name/SKILL.md
  Block hook on Bash commands       .claude/settings.json (hooks)
  Remember my name for this project ~/.claude/projects/<id>/memory/
  Enterprise-wide rules (IT)        /etc/claude-code/CLAUDE.md
```

---

## Summary

```
┌─────────────────────────────────────────────────────────────────┐
│  FILE PATHS CHEATSHEET                                          │
├─────────────────────────────────────────────────────────────────┤
│  User settings    : ~/.claude/settings.json                     │
│  Project settings : .claude/settings.json  (commit)            │
│  Local settings   : .claude/settings.local.json  (gitignore)   │
│  User instructions: ~/.claude/CLAUDE.md                         │
│  Project instruct : .claude/CLAUDE.md  (commit)                │
│  Path-scoped rules: .claude/rules/*.md  (commit)               │
│  Skills           : .claude/skills/*/SKILL.md  (commit)        │
│  MCP project      : .mcp.json  (commit)                        │
│  MCP user         : ~/.claude/.mcp.json                         │
│  Memory           : ~/.claude/projects/<id>/memory/             │
│  Transcripts      : ~/.claude/projects/<id>/<session>/          │
│  Managed (macOS)  : /Library/Application Support/ClaudeCode/    │
│  Managed (Linux)  : /etc/claude-code/                           │
└─────────────────────────────────────────────────────────────────┘
```
