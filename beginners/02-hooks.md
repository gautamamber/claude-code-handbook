# 02 — Hooks System: The Deterministic Layer

---

## What Are Hooks?

Claude makes probabilistic decisions. You cannot 100% guarantee it will always do
or not do something through instructions alone.

Hooks are shell scripts (or HTTP endpoints) that fire at specific events.
They run BEFORE or AFTER Claude's actions. They can block actions with hard stops.

```
┌─────────────────────────────────────────────────────────────────┐
│          INSTRUCTIONS vs HOOKS                                  │
│                                                                 │
│  Instruction:  "Never run rm -rf"                               │
│  Reality:      Claude usually follows it. Not guaranteed.       │
│                                                                 │
│  Hook (exit 2): fires before every Bash call                    │
│  Reality:       rm -rf is physically blocked. No exceptions.    │
└─────────────────────────────────────────────────────────────────┘
```

---

## Hook Lifecycle — Full Event Map

```
USER TYPES MESSAGE
        │
        ▼
┌───────────────────┐
│  UserPromptSubmit │  ← fires before Claude even reads your message
└───────────────────┘
        │
        ▼
   Claude thinks...
        │
        ▼
┌───────────────────┐
│   PreToolUse      │  ← fires before EVERY tool call (Bash, Edit, Read, etc.)
└───────────────────┘
        │
    ┌───┴────────────────┐
    │ blocked (exit 2)?  │
    └───┬────────────────┘
        │ No              │ Yes
        ▼                 ▼
┌───────────────────┐  stderr sent to Claude as feedback
│ PermissionRequest │  Claude adjusts or stops
└───────────────────┘
        │
        ▼
   Tool executes
        │
    ┌───┴──────────────┐
    │  Success or fail? │
    └───┬──────────────┘
        │ Success          │ Fail
        ▼                  ▼
┌────────────────┐  ┌──────────────────────┐
│  PostToolUse   │  │  PostToolUseFailure   │
└────────────────┘  └──────────────────────┘
        │
        ▼
   Claude responds
        │
        ▼
┌───────────────────┐
│      Stop         │  ← Claude finished a full response turn
└───────────────────┘
        │
        ▼
┌───────────────────┐
│    SessionEnd     │  ← session closes
└───────────────────┘
```

---

## All Hook Events

```
SESSION LIFECYCLE
  SessionStart              session begins or resumes
  SessionEnd                session terminates

INSTRUCTION LOADING
  InstructionsLoaded        any CLAUDE.md / rules file loads into context

USER INPUT
  UserPromptSubmit          user submits a prompt (before Claude processes)

TOOL EXECUTION
  PreToolUse                before any tool runs            ← most powerful
  PermissionRequest         permission dialog would appear
  PostToolUse               tool succeeded
  PostToolUseFailure        tool failed

CONTEXT
  PreCompact                before context compaction
  PostCompact               after context compaction

CLAUDE RESPONSE
  Stop                      Claude finishes responding
  StopFailure               turn ended due to API error

SUBAGENTS
  SubagentStart             subagent spawned
  SubagentStop              subagent finished

TASKS
  TaskCreated               background task created
  TaskCompleted             background task finished

WORKTREES
  WorktreeCreate            git worktree created
  WorktreeRemove            git worktree removed

MISC
  TeammateIdle              agent team member about to idle
  Notification              Claude Code sends a notification
  ConfigChange              settings file changed during session
  CwdChanged                working directory changed
  FileChanged               watched file changed
  Elicitation               MCP server requests user input
  ElicitationResult         user responded to MCP elicitation
```

---

## Hook Types

```
┌──────────────────────────────────────────────────────────────────┐
│  TYPE: command                                                   │
│  Runs a local shell script.                                      │
│  Receives JSON on stdin. Responds via stdout/stderr + exit code. │
│                                                                  │
│  "type": "command",                                              │
│  "command": "~/.claude/hooks/check-bash.sh"                     │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│  TYPE: http                                                      │
│  HTTP POST to an endpoint with JSON body.                        │
│  Response determines allow/deny.                                 │
│                                                                  │
│  "type": "http",                                                 │
│  "url": "http://localhost:9000/hook"                             │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│  TYPE: prompt                                                    │
│  Sends the action to Claude Haiku for a yes/no decision.         │
│  Single-turn evaluation. Fast and cheap.                         │
│                                                                  │
│  "type": "prompt",                                               │
│  "prompt": "Is this Bash command safe to run?"                   │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│  TYPE: agent                                                     │
│  Full multi-turn subagent evaluates the action.                  │
│  Default: 60s timeout, up to 50 turns.                          │
│  Use for complex policy checks.                                  │
│                                                                  │
│  "type": "agent",                                                │
│  "agent": "security-reviewer"                                    │
└──────────────────────────────────────────────────────────────────┘
```

---

## Exit Code Semantics — The Most Important Detail

```
┌─────────────────────────────────────────────────────────────────┐
│                     EXIT CODE BEHAVIOR                          │
├──────────────┬──────────────────────────────────────────────────┤
│  exit 0      │  ALLOW the action                                │
│              │  stdout content → added to Claude's context      │
│              │  (Claude sees what you print to stdout)          │
├──────────────┼──────────────────────────────────────────────────┤
│  exit 2      │  BLOCK the action (hard stop)                    │
│              │  stderr content → sent to Claude as feedback     │
│              │  (Claude reads your error message and adjusts)   │
├──────────────┼──────────────────────────────────────────────────┤
│  other code  │  Continue (action proceeds)                      │
│              │  stderr is logged (visible with Ctrl+O)          │
│              │  stdout ignored                                  │
└──────────────┴──────────────────────────────────────────────────┘
```

### Practical Example — Blocking rm -rf

```bash
#!/bin/bash
# ~/.claude/hooks/check-bash.sh
# Reads JSON from stdin, checks for dangerous commands

INPUT=$(cat)
COMMAND=$(echo "$INPUT" | jq -r '.tool_input.command // ""')

if echo "$COMMAND" | grep -qE 'rm\s+-rf|rm\s+--recursive.*-f'; then
  echo "Blocked: rm -rf is not allowed. Use trash or move to /tmp instead." >&2
  exit 2
fi

exit 0
```

Flow:
```
Claude wants to run: rm -rf ./dist

         │
         ▼
  PreToolUse hook fires
         │
         ▼
  check-bash.sh receives JSON:
  {
    "tool_name": "Bash",
    "tool_input": { "command": "rm -rf ./dist" }
  }
         │
         ▼
  grep matches "rm -rf"
         │
         ▼
  echo "Blocked: ..." >&2
  exit 2
         │
         ▼
  Claude receives:
  "Blocked: rm -rf is not allowed. Use trash or move to /tmp instead."
         │
         ▼
  Claude tries alternative approach
```

---

## Hook JSON Input Schema

Every hook receives this JSON on stdin:

```json
{
  "session_id": "abc123",
  "transcript_path": "/Users/alice/.claude/projects/myapp/transcript.jsonl",
  "cwd": "/Users/alice/work/myapp",
  "permission_mode": "default",
  "hook_event_name": "PreToolUse",
  "tool_name": "Bash",
  "tool_input": {
    "command": "npm test"
  },
  "agent_id": null,
  "agent_type": null
}
```

---

## Hook JSON Output (for allow/deny decisions)

Your script can print structured JSON to stdout (for `exit 0`) to influence Claude:

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "allow",
    "permissionDecisionReason": "Safe read-only command",
    "additionalContext": "Command verified against allowlist"
  }
}
```

For blocking (exit 2), just write to stderr — no JSON needed:
```bash
echo "This command is not allowed: $REASON" >&2
exit 2
```

---

## Hook Configuration in settings.json

```json
{
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
    ],
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "cd $CLAUDE_PROJECT_DIR && npx eslint --fix",
            "async": true
          }
        ]
      }
    ],
    "UserPromptSubmit": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/hooks/log-session.sh"
          }
        ]
      }
    ]
  }
}
```

---

## Hook Matchers Reference

Matchers are **case-sensitive**. Use `|` for OR logic.

```
┌─────────────────────┬────────────────────────────────────────────┐
│  Event              │  Matcher examples                          │
├─────────────────────┼────────────────────────────────────────────┤
│  PreToolUse         │  Bash                                      │
│  PostToolUse        │  Edit|Write                                │
│  PostToolUseFailure │  mcp__.*          (all MCP tools)          │
│  PermissionRequest  │  mcp__github__.*  (GitHub MCP only)        │
├─────────────────────┼────────────────────────────────────────────┤
│  SessionStart       │  startup                                   │
│  SessionEnd         │  resume  |  clear                          │
├─────────────────────┼────────────────────────────────────────────┤
│  Notification       │  permission_prompt  |  auth_success        │
├─────────────────────┼────────────────────────────────────────────┤
│  ConfigChange       │  user_settings  |  project_settings        │
├─────────────────────┼────────────────────────────────────────────┤
│  FileChanged        │  .env  |  .envrc                           │
├─────────────────────┼────────────────────────────────────────────┤
│  StopFailure        │  rate_limit  |  authentication_failed      │
├─────────────────────┼────────────────────────────────────────────┤
│  InstructionsLoaded │  session_start  |  nested_traversal        │
└─────────────────────┴────────────────────────────────────────────┘
```

---

## Environment Variables in Hooks

```bash
$CLAUDE_PROJECT_DIR      # /Users/alice/work/myapp
$CLAUDE_PLUGIN_ROOT      # plugin installation directory
$CLAUDE_ENV_FILE         # path to file for persisting env vars
                         # (available in SessionStart, CwdChanged, FileChanged)
$CLAUDE_CODE_REMOTE      # "true" when running in cloud environment
```

---

## Async Hooks

Normal hooks block the action until they complete.
Async hooks run in the background — action proceeds immediately.

```json
{
  "type": "command",
  "command": "~/.claude/hooks/audit-log.sh",
  "async": true
}
```

```
Without async:                    With async:

  Claude wants to edit file         Claude wants to edit file
          │                                 │
          ▼                                 ▼
    Hook fires                    Hook fires (background)──────────────►
          │                                 │                           runs later
    Waits for hook                    Edit happens NOW
          │
    Edit happens
```

Use async for logging, notifications, audit trails — anything that shouldn't block Claude.

---

## Hook Timeout

Default timeout: 10 minutes. Configure per hook:

```json
{
  "type": "command",
  "command": "~/.claude/hooks/slow-check.sh",
  "timeout": 30
}
```

`timeout` is in seconds.

---

## Hook Deduplication

If multiple hooks with identical commands run in parallel for the same event,
Claude Code automatically deduplicates them — the command runs only once.

---

## Where to Store Hook Config

```
~/.claude/settings.json          applies to ALL projects on this machine
.claude/settings.json            applies to THIS project (shareable via git)
.claude/settings.local.json      applies to THIS project (local only, gitignored)
```

---

## Real-World Hook Use Cases

```
┌──────────────────────────────────────────────────────────────────┐
│  SECURITY                                                        │
│  • Block dangerous commands (rm -rf, curl | bash, etc.)         │
│  • Prevent pushing to main branch                               │
│  • Block sending secrets to external endpoints                  │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│  CODE QUALITY                                                    │
│  • Run linter after every file edit                             │
│  • Run formatter after Write                                    │
│  • Run tests after code changes                                 │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│  AUDIT & LOGGING                                                 │
│  • Log all commands Claude runs (async hook)                    │
│  • Record session transcripts                                   │
│  • Send notifications to Slack on task completion              │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│  CONTEXT INJECTION                                               │
│  • UserPromptSubmit: inject current git branch + last 5 commits │
│  • SessionStart: inject team's current sprint tickets           │
│  • FileChanged: inject fresh .env values when .env changes      │
└──────────────────────────────────────────────────────────────────┘
```

---

## Summary

```
┌─────────────────────────────────────────────────────────────────┐
│  HOOKS CHEATSHEET                                               │
├─────────────────────────────────────────────────────────────────┤
│  Types       : command | http | prompt | agent                  │
│  exit 0      : allow — stdout adds to Claude's context          │
│  exit 2      : BLOCK — stderr becomes Claude's feedback         │
│  async: true : run in background, don't block action            │
│  timeout     : seconds (default 600 / 10 min)                   │
│  Matchers    : case-sensitive, supports | for OR, .* for regex  │
│  Config      : hooks key inside settings.json                   │
│  Strongest   : PreToolUse + exit 2 = hard enforcement           │
└─────────────────────────────────────────────────────────────────┘
```
