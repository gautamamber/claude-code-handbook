# Expert 05 — Debugging & Diagnostics

---

## Diagnostic Arsenal

```
  ┌──────────────────────────────────────────────────────────────┐
  │  Tool              │  What It Shows                         │
  ├──────────────────────────────────────────────────────────────┤
  │  /doctor           │  Auth, paths, keybindings, version     │
  │  /status           │  Current model, auth, session ID       │
  │  /context          │  Context window breakdown by source    │
  │  /cost             │  Token usage and USD cost so far       │
  │  /hooks            │  Active hooks configuration            │
  │  /mcp              │  MCP server connections and status     │
  │  Ctrl+O            │  Toggle verbose transcript view        │
  │  --debug           │  Low-level API and system logs         │
  │  --verbose         │  Full turn-by-turn tool execution      │
  └──────────────────────────────────────────────────────────────┘
```

---

## /doctor — The First Thing to Run

When something is broken, run `/doctor` first.

```
  /doctor checks:
  ┌──────────────────────────────────────────────────────────┐
  │  ✓ Claude Code version: 1.x.x                           │
  │  ✓ Authentication: logged in as user@example.com        │
  │  ✓ API connectivity: Anthropic API reachable            │
  │  ✓ Node.js version: 20.x.x                              │
  │  ✓ PATH: claude command found at /usr/local/bin/claude  │
  │  ✓ Keybindings: no conflicts found                      │
  │  ✗ MCP server "github": connection timeout              │
  │  ✗ Hook script "check-bash.sh": file not found          │
  └──────────────────────────────────────────────────────────┘
```

---

## Debug Logging

```bash
# All debug categories
claude --debug

# Specific categories
claude --debug "api"          # API request/response details
claude --debug "mcp"          # MCP server communication
claude --debug "hooks"        # Hook execution logs
claude --debug "file"         # File operation logs

# Multiple categories
claude --debug "api,mcp"

# Exclude specific categories (useful when all is too noisy)
claude --debug "!statsig,!file"
```

### What --debug api Shows

```
  [API] → POST /v1/messages
  [API]   model: claude-sonnet-4-6
  [API]   max_tokens: 8096
  [API]   tools: 12 tools
  [API]   messages: 4 messages (12,456 tokens)

  [API] ← 200 OK (2.3s)
  [API]   stop_reason: tool_use
  [API]   usage: {input: 12456, output: 234, cache_read: 8900}
  [API]   cost: $0.0189

  [API] Tool: Read
  [API]   input: {file_path: "src/auth/token.ts"}
  [API]   duration: 12ms
  [API]   output_tokens: 456

  [API] → POST /v1/messages (turn 2)
  ...
```

---

## Verbose Mode

```bash
claude --verbose
# Or toggle during session with Ctrl+O
```

Shows every tool call with inputs and outputs:

```
  ─── Turn 1 ─────────────────────────────────────────────────────
  Claude: I'll look at the auth file first.

  ┌─ Tool: Read ────────────────────────────────────────────────┐
  │ Input: { file_path: "src/auth/token.ts" }                   │
  │ Duration: 8ms                                                │
  │ Output: 47 lines                                             │
  └─────────────────────────────────────────────────────────────┘

  ─── Turn 2 ─────────────────────────────────────────────────────
  Claude: I can see the issue. Let me fix it.

  ┌─ Tool: Edit ────────────────────────────────────────────────┐
  │ Input: { file_path: "src/auth/token.ts",                    │
  │          old_string: "expiresIn: '1h'",                     │
  │          new_string: "expiresIn: '24h'" }                   │
  │ Duration: 5ms                                               │
  │ Output: Success                                              │
  └─────────────────────────────────────────────────────────────┘
```

---

## Transcript Inspection

Every session writes a transcript. Inspect it to understand exactly what happened.

```
  Location:
  ~/.claude/projects/<project-id>/<session-id>/transcript.jsonl

  Find your session:
  ls ~/.claude/projects/
  ls ~/.claude/projects/<project-id>/
```

```bash
# Pretty-print the transcript
cat ~/.claude/projects/*/transcript.jsonl | jq '.'

# Find all tool calls
cat transcript.jsonl | jq 'select(.type == "tool_use")'

# Find all errors
cat transcript.jsonl | jq 'select(.is_error == true)'

# Find the most expensive turns
cat transcript.jsonl | jq 'select(.usage) | {tokens: .usage.output_tokens}' | sort -n

# Extract just Claude's responses
cat transcript.jsonl | jq 'select(.role == "assistant") | .content[].text // empty'

# Find all file edits
cat transcript.jsonl | jq 'select(.type=="tool_use" and .name=="Edit") | .input.file_path'
```

---

## Common Problems and Fixes

### Problem: Claude repeatedly fails at a task

```
  Symptom: Claude tries the same approach 3+ times and fails

  Diagnose:
  1. Run Ctrl+O to see full tool outputs
  2. Find the actual error in tool_result
  3. Check: is it a permission issue? Wrong path? Missing dep?

  Fix:
  • Give Claude the actual error context
  • Or use /rewind to go back and redirect approach
  • Or start fresh with a more specific prompt
```

### Problem: Hook not firing

```
  Symptom: Expected hook doesn't run

  Diagnose:
  1. Run /hooks to see active hooks
  2. Check matcher case: "Bash" not "bash"
  3. Check settings.json syntax: run jq . .claude/settings.json
  4. Check hook script exists: ls path/to/hook.sh
  5. Check hook script is executable: chmod +x hook.sh

  Fix:
  • Correct the matcher (case-sensitive)
  • Fix JSON syntax in settings.json
  • Make script executable
```

### Problem: MCP server not connecting

```
  Symptom: "mcp__github__*" tool not available or failing

  Diagnose:
  1. Run /mcp to see server status
  2. Check if server process started: ps aux | grep mcp
  3. Check MCP_TIMEOUT env var (default too low?)
  4. Run: claude mcp get <name>

  Fix:
  • Increase timeout: export MCP_TIMEOUT=30000
  • Fix command in .mcp.json
  • Check env vars are set (GITHUB_TOKEN, etc.)
  • Run --debug mcp for full communication log
```

### Problem: Context window fills too fast

```
  Symptom: Compaction happens after 5-10 turns

  Diagnose:
  1. Run /context to see breakdown
  2. Identify what's consuming space:
     - Large file reads?
     - Verbose bash output?
     - Many MCP tool calls?
     - Extended thinking tokens?

  Fix:
  • Ask Claude to summarize file contents instead of reading all
  • Pipe bash output through grep/head before returning
  • Use subagents for file-heavy research
  • Disable extended thinking for simple tasks
```

### Problem: Edit tool keeps failing with "old_string not found"

```
  Symptom: Claude tries to edit but old_string never matches

  Diagnose:
  1. Check if file was modified externally since Claude read it
  2. Check for invisible characters (tabs vs spaces)
  3. Check for trailing whitespace differences
  4. Claude may have constructed old_string from memory, not file

  Fix:
  • Tell Claude to re-read the file before editing
  • Or use Write tool to rewrite entire file section
  • Or give Claude the exact current file contents
```

### Problem: "Permission denied" or "file not accessible"

```
  Symptom: Claude can't read/write a file outside working directory

  Diagnose:
  1. Check current working directory
  2. Check sandbox settings
  3. Check if file path is within allowed paths

  Fix:
  • claude --add-dir /path/to/other/dir
  • Or update sandbox.paths in settings.json
  • Or start claude from parent directory
```

### Problem: Sessions not resuming correctly

```
  Symptom: --resume loads wrong session or fails

  Diagnose:
  1. Check session exists: ls ~/.claude/projects/<project>/
  2. Check session ID format (must be UUID for --session-id)
  3. Try --resume with session name instead of ID

  Fix:
  • Use full UUID from /session command
  • Or use display name from --name flag
  • Or use -c (continue most recent) instead
```

---

## Context Window Debugging

```
  /context output explained:

  ┌──────────────────────────────────────────────────────────────┐
  │  Context usage: 45,231 / 200,000 tokens (22.6%)             │
  │                                                              │
  │  System prompt:            2,100 tokens  ← fixed overhead   │
  │  CLAUDE.md (project):      1,800 tokens  ← your instructions│
  │  CLAUDE.md (user):           400 tokens                     │
  │  Memory (MEMORY.md):         950 tokens                     │
  │  Skill descriptions (12):    800 tokens                     │
  │  MCP tool names:             320 tokens  ← names only       │
  │                                                              │
  │  Conversation history:    18,500 tokens  ← grows each turn  │
  │  Tool results:            21,000 tokens  ← can be large     │
  │    ├── Read (token.ts):    1,200 tokens                     │
  │    ├── Bash (npm test):    4,500 tokens  ← verbose output   │
  │    └── Read (auth.ts):     1,100 tokens                     │
  │                                                              │
  │  MCP server tool defs:     2,500 tokens  ← loaded tools     │
  │    ├── github (3 tools):   1,800 tokens                     │
  │    └── slack (1 tool):       700 tokens                     │
  └──────────────────────────────────────────────────────────────┘

  Quick actions from this info:
  → "Tool results: 21k" is large → use /compact
  → "Bash (npm test): 4.5k" → pipe through tail -n 50 next time
  → "MCP tool defs: 2.5k" → remove unused MCP servers
```

---

## Hook Debugging

```bash
# Test a hook manually before relying on it
echo '{"tool_name":"Bash","tool_input":{"command":"rm -rf /tmp/test"}}' | \
  ~/.claude/hooks/check-bash.sh

# If exit 0: hook would allow
# If exit 2: hook would block

# Check what hook receives by logging it
cat > /tmp/debug-hook.sh << 'EOF'
#!/bin/bash
INPUT=$(cat)
echo "Hook received: $INPUT" >> /tmp/hook-debug.log
exit 0
EOF
chmod +x /tmp/debug-hook.sh

# Add to settings.json temporarily
# "command": "/tmp/debug-hook.sh"
# Then check /tmp/hook-debug.log after running Claude
```

---

## Performance Profiling

```bash
# How long does each turn take?
claude --debug api "complex task" 2>&1 | grep -E "duration|elapsed|ms"

# How much does each session cost?
# At end of session:
/cost

# Breakdown by tool type:
/cost --detailed   # (if available in your version)

# For CI: capture cost in JSON output
claude -p "task" --output-format json | jq '.cost_usd'
```

---

## Summary

```
┌────────────────────────────────────────────────────────────────┐
│  DEBUGGING CHEATSHEET                                          │
├────────────────────────────────────────────────────────────────┤
│  First step      : /doctor (checks auth, paths, hooks, MCP)    │
│  See tool calls  : Ctrl+O or --verbose                         │
│  API logs        : --debug api                                 │
│  MCP issues      : --debug mcp, check /mcp status             │
│  Hook issues     : /hooks, check matcher case, test manually   │
│  Context full    : /context → find what's consuming space      │
│  Edit fails      : tell Claude to re-read file before editing  │
│  Session issues  : check ~/.claude/projects/<id>/              │
│  Transcript      : transcript.jsonl → jq for analysis         │
│  Permission err  : --add-dir, sandbox.paths in settings        │
└────────────────────────────────────────────────────────────────┘
```
