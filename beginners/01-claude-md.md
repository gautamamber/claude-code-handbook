# 01 — CLAUDE.md: Context, Not Config

---

## What Is CLAUDE.md?

CLAUDE.md is an instruction file Claude reads at the start of every session.
Most people think it works like a config file. It does not.

```
┌─────────────────────────────────────────────────────────┐
│                  COMMON MISCONCEPTION                   │
│                                                         │
│   CLAUDE.md  ──►  system prompt  ──►  enforced rule     │
│                                                         │
│                  ACTUAL REALITY                         │
│                                                         │
│   CLAUDE.md  ──►  user message  ──►  persuasive hint    │
└─────────────────────────────────────────────────────────┘
```

Claude reads CLAUDE.md the same way it reads your messages.
It can follow, misinterpret, or (if context gets large) even forget what was in it.

---

## The Scope Hierarchy

There are 4 levels. Higher levels have higher priority.

```
PRIORITY
  HIGH
   │
   │   ┌──────────────────────────────────────────────────────────┐
   │   │  MANAGED POLICY                                          │
   │   │  /Library/Application Support/ClaudeCode/CLAUDE.md      │  ← macOS
   │   │  /etc/claude-code/CLAUDE.md                             │  ← Linux
   │   │                                                          │
   │   │  Set by IT / enterprise. Cannot be excluded or ignored.  │
   │   └──────────────────────────────────────────────────────────┘
   │
   │   ┌──────────────────────────────────────────────────────────┐
   │   │  PROJECT                                                 │
   │   │  ./.claude/CLAUDE.md   or   ./CLAUDE.md                 │
   │   │                                                          │
   │   │  Shared with your team via git.                         │
   │   │  Best place for project-specific coding standards.      │
   │   └──────────────────────────────────────────────────────────┘
   │
   │   ┌──────────────────────────────────────────────────────────┐
   │   │  USER                                                    │
   │   │  ~/.claude/CLAUDE.md                                     │
   │   │                                                          │
   │   │  Your personal preferences across all projects.         │
   │   │  Tone, style, habits you always want Claude to follow.  │
   │   └──────────────────────────────────────────────────────────┘
   │
  LOW
```

---

## How Claude Finds and Loads CLAUDE.md

Claude walks UP the directory tree from your current working directory.

```
Your cwd:   /Users/alice/work/myapp/src/components/

Walk-up sequence:
  /Users/alice/work/myapp/src/components/CLAUDE.md   ← checked
  /Users/alice/work/myapp/src/CLAUDE.md              ← checked
  /Users/alice/work/myapp/CLAUDE.md                  ← FOUND ✓ loaded
  /Users/alice/work/CLAUDE.md                        ← checked
  /Users/alice/CLAUDE.md                             ← checked
  ~/.claude/CLAUDE.md                                ← always loaded last

All found files are loaded. All scopes apply simultaneously.
```

---

## Lazy Loading — Subdirectory Files

Project-level and ancestor files load at session start.
Files in subdirectories load only when Claude reads files in that directory.

```
Session Start
     │
     ▼
┌────────────────────────────────────────────────┐
│  Loaded immediately at startup                 │
│  • managed policy CLAUDE.md                    │
│  • project CLAUDE.md (./.claude/CLAUDE.md)     │
│  • user CLAUDE.md (~/.claude/CLAUDE.md)        │
└────────────────────────────────────────────────┘
           │
           │  Later, when Claude reads src/api/server.ts
           ▼
┌────────────────────────────────────────────────┐
│  Loaded on demand (lazy)                       │
│  • src/api/CLAUDE.md     ← if it exists        │
│  • src/CLAUDE.md         ← if it exists        │
└────────────────────────────────────────────────┘
```

---

## File Imports with @

You can split instructions across multiple files using `@path/to/file` imports.

```
.claude/CLAUDE.md
┌──────────────────────────────────────────────┐
│  # Project Standards                         │
│                                              │
│  @.claude/standards/typescript.md           │  ─── imports ──►  .claude/standards/typescript.md
│  @.claude/standards/testing.md             │  ─── imports ──►  .claude/standards/testing.md
│  @.claude/standards/git.md                 │  ─── imports ──►  .claude/standards/git.md
└──────────────────────────────────────────────┘

Rules:
  • Paths resolve relative to the importing file (not cwd)
  • Maximum import depth: 5 hops
  • Circular imports are detected and stopped
```

---

## Size Limits

```
┌──────────────────────────────────────────────────────┐
│                 CLAUDE.md SIZE LIMIT                 │
│                                                      │
│         200 lines   OR   25 KB                      │
│         (whichever comes first)                      │
│                                                      │
│  ████████████████████████░░░░  ← 200 lines = limit  │
│  ░░░░░░░░░░░░░░░░░░░░░░░░░░░░  ← rest is ignored    │
│                                                      │
│  Keep CLAUDE.md concise. Put details in sub-files   │
│  and import them with @path/to/file                 │
└──────────────────────────────────────────────────────┘
```

---

## HTML Comments Are Stripped

You can add maintainer-only notes that Claude never sees.

```markdown
<!-- This section added by Alice on 2025-01-10 — do not remove -->
Always use snake_case for Python variable names.
<!-- TODO: add more examples once team agrees -->
```

What Claude receives:
```
Always use snake_case for Python variable names.
```

HTML comments are stripped before injection into context.

---

## Path-Scoped Rules — .claude/rules/

Instead of putting everything in CLAUDE.md, you can create rule files
that apply only to specific file paths.

```
.claude/
└── rules/
    ├── typescript.md        ← applies to: src/**/*.ts
    ├── python.md            ← applies to: **/*.py
    └── tests.md             ← applies to: **/*.test.*

Each rule file has YAML frontmatter:
```

```markdown
---
paths:
  - "src/**/*.ts"
  - "src/**/*.tsx"
---

Always use `interface` over `type` for object shapes.
Use `const` assertions for literal types.
Never use `any` — use `unknown` instead.
```

```
Without paths: frontmatter → loads at session start (same as CLAUDE.md)
With    paths: frontmatter → loads only when Claude reads a matching file
```

---

## Excluding Specific CLAUDE.md Files

In settings, you can prevent specific CLAUDE.md files from loading.

```json
{
  "claudeMdExcludes": [
    "**/legacy/CLAUDE.md",
    "**/vendor/CLAUDE.md"
  ]
}
```

```
NOTE: The managed policy CLAUDE.md can never be excluded.
```

---

## How Claude.md Flows Into a Session

```
                  Session Start
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
   Managed policy   Project      User
   CLAUDE.md        CLAUDE.md    CLAUDE.md
          │            │            │
          └────────────┴────────────┘
                       │
                       ▼
              All content merged
              into a user message
                       │
                       ▼
          ┌────────────────────────┐
          │  Claude's Context      │
          │  ┌──────────────────┐  │
          │  │  System prompt   │  │  ← Claude's hardcoded behavior
          │  ├──────────────────┤  │
          │  │  CLAUDE.md       │  │  ← Your instructions (as user msg)
          │  ├──────────────────┤  │
          │  │  Conversation    │  │
          │  └──────────────────┘  │
          └────────────────────────┘
```

---

## Good vs Bad Instructions

Because CLAUDE.md is a user message (not enforced config), instruction quality matters.

```
BAD — Too vague:
  "Write good code."
  "Be careful with files."
  "Follow best practices."

GOOD — Specific and actionable:
  "Never use var — always use const or let."
  "Before deleting any file, confirm with the user."
  "All SQL queries must use parameterized statements, never string concatenation."
```

Vague instructions fail silently. Claude follows the spirit of what it can interpret.
Specific instructions produce consistent behavior.

---

## Summary

```
┌─────────────────────────────────────────────────────────────────┐
│  CLAUDE.md CHEATSHEET                                           │
├─────────────────────────────────────────────────────────────────┤
│  Type          : User message (not system prompt)               │
│  Size limit    : 200 lines OR 25 KB                             │
│  Import syntax : @path/to/other-file.md (max 5 hops)           │
│  Comments      : <!-- html comments --> stripped before loading │
│  Subdirs       : Loaded lazily (on file access, not at startup) │
│  Scoped rules  : .claude/rules/*.md with paths: frontmatter     │
│  Exclusion     : claudeMdExcludes in settings (not managed)     │
│  Priority      : managed > project > user                       │
└─────────────────────────────────────────────────────────────────┘
```
