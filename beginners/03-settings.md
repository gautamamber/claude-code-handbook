# 03 — Settings: Precedence and Merge Behavior

---

## Overview

Claude Code has a layered settings system. Multiple files can define the same setting.
Understanding which file wins — and how arrays merge — is critical for team setups.

---

## The 4 Scopes (Highest → Lowest Priority)

```
┌─────────────────────────────────────────────────────────────────────┐
│  SCOPE 1 — MANAGED POLICY  (highest priority, cannot be overridden) │
│                                                                     │
│  macOS:    /Library/Application Support/ClaudeCode/managed.json    │
│  Linux:    /etc/claude-code/managed-settings.json                  │
│  Windows:  C:\Program Files\ClaudeCode\managed-settings.json       │
│                                                                     │
│  Set by IT/enterprise administrators.                               │
│  Overrides everything below it. Users cannot change it.             │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              │ lower priority
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│  SCOPE 2 — LOCAL PROJECT                                            │
│                                                                     │
│  .claude/settings.local.json                                        │
│                                                                     │
│  Per-machine, per-project settings.                                 │
│  Should be in .gitignore — never committed.                         │
│  Your local overrides: model choice, tool allowances, etc.          │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│  SCOPE 3 — SHARED PROJECT                                           │
│                                                                     │
│  .claude/settings.json                                              │
│                                                                     │
│  Committed to git. Shared with the whole team.                      │
│  Use for: project permission rules, hook configs, MCP servers.      │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│  SCOPE 4 — USER (lowest priority)                                   │
│                                                                     │
│  ~/.claude/settings.json                                            │
│                                                                     │
│  Your personal settings, applied to every project.                  │
│  Use for: default model, personal tool preferences.                 │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Merge Behavior — Primitives vs Arrays

This is the part most people misunderstand.

```
┌──────────────────────────────────────────────────────────────────┐
│  PRIMITIVES — highest priority wins, lower scopes ignored        │
│                                                                  │
│  ~/.claude/settings.json:                                        │
│    "defaultModel": "sonnet"                                      │
│                                                                  │
│  .claude/settings.json:                                          │
│    "defaultModel": "opus"           ← this wins (higher scope)  │
│                                                                  │
│  Result: "opus"                                                  │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│  ARRAYS — all scopes merge together                              │
│                                                                  │
│  ~/.claude/settings.json:                                        │
│    "permissions": { "allow": ["Bash(git *)"] }                  │
│                                                                  │
│  .claude/settings.json:                                          │
│    "permissions": { "allow": ["Bash(npm *)"] }                  │
│                                                                  │
│  Result: allow = ["Bash(git *)", "Bash(npm *)"]   ← BOTH merged │
│                                                                  │
│  Same for deny arrays and hooks arrays.                          │
└──────────────────────────────────────────────────────────────────┘
```

---

## All Settings Files at a Glance

```
~/.claude/
├── settings.json             ← user settings (all projects)
├── keybindings.json          ← global key bindings
└── .mcp.json                 ← user MCP server definitions

[project root]/
├── .claude/
│   ├── settings.json         ← project shared settings (commit this)
│   ├── settings.local.json   ← project local settings (gitignore this)
│   ├── keybindings.local.json
│   ├── CLAUDE.md             ← project instructions
│   ├── rules/                ← path-scoped rule files
│   ├── skills/               ← project slash commands
│   └── agents/               ← project subagent definitions
└── .mcp.json                 ← project MCP servers (commit this)
```

---

## Full Settings Schema

```json
{
  // ── PERMISSIONS ─────────────────────────────────────────────
  "permissions": {
    "defaultMode": "default",
    // Options: default | acceptEdits | plan | auto | dontAsk | bypassPermissions

    "allow": [
      "Bash(npm test)",           // allow exactly 'npm test'
      "Bash(npm *)",              // allow any npm command
      "Edit(*.ts)",               // allow editing .ts files
      "Bash(git *)"               // allow all git commands
    ],
    "deny": [
      "Bash(rm -rf *)",
      "Bash(curl * | bash)",
      "Bash(sudo *)"
    ],
    "ask": [
      "Bash(git push *)"          // always prompt before pushing
    ]
  },

  // ── SANDBOX ─────────────────────────────────────────────────
  "sandbox": {
    "enabled": true,
    "paths": [
      "/additional/accessible/path"
    ]
  },

  // ── HOOKS ───────────────────────────────────────────────────
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/hooks/check-bash.sh",
            "timeout": 10
          }
        ]
      }
    ]
  },

  // ── CLAUDE.MD CONTROL ───────────────────────────────────────
  "claudeMdExcludes": [
    "**/vendor/CLAUDE.md",
    "**/legacy/CLAUDE.md"
  ],

  // ── MEMORY ──────────────────────────────────────────────────
  "autoMemoryEnabled": true,
  "autoMemoryDirectory": "~/custom-memory-dir",

  // ── MODEL ───────────────────────────────────────────────────
  "defaultModel": "sonnet",
  // Options: sonnet | opus | haiku

  // ── HOOKS KILL SWITCH ───────────────────────────────────────
  "disableAllHooks": false,

  // ── PLUGINS ─────────────────────────────────────────────────
  "plugins": {
    "enabledPlugins": ["my-plugin"],
    "extraKnownMarketplaces": {
      "my-marketplace": {
        "source": {
          "source": "github",
          "repo": "owner/repo"
        }
      }
    }
  }
}
```

---

## Permission Rule Syntax

```
Format:   ToolName(pattern)

Examples:
  Bash(npm test)            exact command match
  Bash(npm *)               npm + anything after
  Bash(git *)               all git commands
  Edit(*.ts)                editing any .ts file
  Edit(src/*)               editing anything in src/
  Read(*.env)               reading .env files
  mcp__github__.*           any GitHub MCP tool
  Agent(my-agent)           specific subagent
```

---

## Typical Team Setup

```
Repository structure:

.claude/
├── settings.json               ← committed to git
│   {
│     "permissions": {
│       "allow": ["Bash(npm *)", "Bash(git *)"],
│       "deny": ["Bash(rm -rf *)"]
│     },
│     "hooks": { ... }
│   }
│
└── settings.local.json         ← in .gitignore
    {
      "permissions": {
        "defaultMode": "auto"   ← developer's personal preference
      },
      "defaultModel": "opus"    ← personal model choice
    }

~/.claude/settings.json         ← personal, applies to all projects
  {
    "permissions": {
      "allow": ["Bash(git *)"]  ← merged with project allow list
    }
  }

FINAL MERGED RESULT:
  allow = ["Bash(npm *)", "Bash(git *)", "Bash(git *)"]  ← deduped
  deny  = ["Bash(rm -rf *)"]
  defaultMode = "auto"   ← from settings.local.json (higher priority than shared)
  defaultModel = "opus"  ← from settings.local.json (higher priority than user)
```

---

## .gitignore Recommendation

```gitignore
# Claude Code local settings (machine-specific)
.claude/settings.local.json
.claude/keybindings.local.json

# Commit these:
# .claude/settings.json     ← shared team rules
# .mcp.json                 ← shared MCP servers
# .claude/CLAUDE.md         ← project instructions
```

---

## Key Differences: settings.json vs CLAUDE.md

```
┌──────────────────────────┬───────────────────────────────────────────┐
│  settings.json           │  CLAUDE.md                                │
├──────────────────────────┼───────────────────────────────────────────┤
│  Machine-readable config │  Human-readable instructions              │
│  Hard enforcement        │  Persuasive guidance                      │
│  Permission rules        │  Coding standards                         │
│  Hook configuration      │  Style preferences                        │
│  Sandbox rules           │  Workflow descriptions                    │
│  Model selection         │  Architecture notes                       │
│  Merged across scopes    │  Priority order, all loaded               │
└──────────────────────────┴───────────────────────────────────────────┘
```

---

## Summary

```
┌─────────────────────────────────────────────────────────────────┐
│  SETTINGS CHEATSHEET                                            │
├─────────────────────────────────────────────────────────────────┤
│  Priority order  : managed > local > project > user             │
│  Primitives      : highest scope wins                           │
│  Arrays          : all scopes merge (allow, deny, hooks)        │
│  Commit this     : .claude/settings.json, .mcp.json             │
│  Gitignore this  : .claude/settings.local.json                  │
│  User settings   : ~/.claude/settings.json                      │
│  Kill switch     : "disableAllHooks": true                       │
└─────────────────────────────────────────────────────────────────┘
```
