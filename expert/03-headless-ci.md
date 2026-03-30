# Expert 03 — Headless & CI Mode: Automating Claude Code

---

## What Is Headless Mode?

The `-p` (print) flag makes Claude Code non-interactive.
No UI. No prompts. Just input → output.

```
  Interactive mode (default):
  ┌────────────────────────────────────┐
  │  > You: Fix the auth bug           │
  │  Claude: I'll look at auth.ts...   │
  │  [streams response in real-time]   │
  │  [asks permission for file edits]  │
  └────────────────────────────────────┘

  Headless mode (-p):
  $ claude -p "Fix the auth bug" --allowedTools "Read,Edit,Bash"
  Fixed the auth bug. Changed expiresIn from '1h' to '24h' in token.ts.
  Tests passing.
  $ echo $?
  0
```

---

## The -p Flag — Core Mechanics

```bash
# Minimum viable headless call
claude -p "your prompt here"

# With auto-approved tools
claude -p "run tests" --allowedTools "Bash"

# With output format
claude -p "analyze code" --output-format json

# With model override
claude -p "quick task" --model haiku

# With turn limit (prevent runaway loops)
claude -p "complex task" --max-turns 10

# With cost limit
claude -p "expensive task" --max-budget-usd 2.00
```

---

## Output Formats

### Plain Text (default)

```bash
claude -p "Summarize src/auth/token.ts"
# Output: Plain text response printed to stdout
```

### JSON

```bash
claude -p "Analyze auth module" --output-format json
```

```json
{
  "result": "The auth module handles JWT token generation...",
  "session_id": "a1b2c3d4-...",
  "usage": {
    "input_tokens": 1234,
    "output_tokens": 567,
    "cache_read_input_tokens": 890
  },
  "cost_usd": 0.0234
}
```

### Streaming JSON (real-time tokens)

```bash
claude -p "write long response" \
  --output-format stream-json \
  --verbose \
  --include-partial-messages
```

Each line is a JSON event:
```json
{"type": "stream_event", "event": {"type": "content_block_delta", "delta": {"type": "text_delta", "text": "Hello"}}}
{"type": "stream_event", "event": {"type": "content_block_delta", "delta": {"type": "text_delta", "text": " world"}}}
{"type": "result", "result": "Hello world", "session_id": "...", "cost_usd": 0.001}
```

Extract just the text tokens:
```bash
claude -p "write essay" \
  --output-format stream-json \
  --include-partial-messages | \
  jq -rj 'select(.type=="stream_event") |
          select(.event.delta.type?=="text_delta") |
          .event.delta.text'
```

---

## Bare Mode — For Reproducible CI

Bare mode skips ALL auto-discovery. Only loads what you explicitly pass.

```bash
claude --bare -p "task" \
  --append-system-prompt "You are a CI bot" \
  --allowedTools "Bash,Read" \
  --settings ./ci-settings.json
```

What `--bare` skips:
```
  ✗ CLAUDE.md files (all scopes)
  ✗ Auto memory (MEMORY.md)
  ✗ Skills discovery
  ✗ Plugins
  ✗ MCP servers from .mcp.json
  ✗ Hooks from settings
```

Use bare mode when:
- You need reproducible, deterministic CI runs
- You don't want project CLAUDE.md affecting CI behavior
- You're testing a specific prompt without ambient context

---

## CI/CD Patterns

### Pattern 1: PR Review Bot

```bash
#!/bin/bash
# .github/workflows/review.sh

PR_DIFF=$(gh pr diff "$PR_NUMBER")

REVIEW=$(echo "$PR_DIFF" | claude -p \
  --bare \
  --model sonnet \
  --max-turns 5 \
  --allowedTools "Read" \
  --append-system-prompt "You are a code reviewer. Be concise and specific." \
  --output-format json \
  "Review this PR diff for bugs, security issues, and code quality.")

COMMENT=$(echo "$REVIEW" | jq -r '.result')
gh pr comment "$PR_NUMBER" --body "$COMMENT"
```

### Pattern 2: Test Failure Analyzer

```bash
#!/bin/bash
# Run tests and auto-fix failures

TEST_OUTPUT=$(npm test 2>&1)
EXIT_CODE=$?

if [ $EXIT_CODE -ne 0 ]; then
  echo "$TEST_OUTPUT" | claude -p \
    --allowedTools "Read,Edit,Bash(npm test)" \
    --max-turns 10 \
    --max-budget-usd 3.00 \
    "These tests are failing. Fix them: $TEST_OUTPUT"
fi
```

### Pattern 3: Multi-Step Pipeline

```bash
#!/bin/bash
# Multi-step CI pipeline using session continuity

# Step 1: Analyze
SESSION_ID=$(claude -p "Analyze the changed files in this PR" \
  --allowedTools "Read,Bash(git diff *)" \
  --output-format json | jq -r '.session_id')

# Step 2: Fix (continue same session)
claude -p "Fix all issues you found" \
  --resume "$SESSION_ID" \
  --allowedTools "Read,Edit,Bash(npm test)" \
  --max-turns 20

# Step 3: Verify (continue same session)
claude -p "Verify all fixes and run final checks" \
  --resume "$SESSION_ID" \
  --allowedTools "Bash(npm test),Bash(npm run lint)" \
  --max-turns 5
```

### Pattern 4: Parallel Analysis Jobs

```bash
#!/bin/bash
# Run multiple analyses in parallel

analyze() {
  MODULE=$1
  claude -p "Analyze $MODULE for security vulnerabilities" \
    --model haiku \
    --allowedTools "Read,Grep,Glob" \
    --max-turns 5 \
    --output-format json > "reports/$MODULE-security.json" &
}

analyze "src/auth"
analyze "src/api"
analyze "src/db"

wait  # Wait for all parallel jobs

# Combine reports
claude -p "$(cat reports/*.json | jq -s .)" \
  --bare \
  --model sonnet \
  "Combine these security reports into a prioritized summary" \
  > combined-report.md
```

---

## Piping Input/Output

### Pipe content to Claude

```bash
# Pipe file content
cat error.log | claude -p "Extract all ERROR and FATAL lines with timestamps"

# Pipe command output
git diff HEAD~1 | claude -p "Summarize these changes for a commit message"

# Pipe JSON
curl -s api.github.com/repos/anthropics/claude-code/issues | \
  claude -p "Find all open bugs related to MCP"
```

### Pipe Claude output to other tools

```bash
# Generate and run code
claude -p "Write a Python script to parse CSV" --bare > parse.py
python parse.py data.csv

# Generate and apply git commit
MESSAGE=$(git diff --staged | claude -p "Write a commit message for these changes" --bare)
git commit -m "$MESSAGE"

# Generate and save as JSON
claude -p "List all API endpoints in this codebase" \
  --allowedTools "Grep,Glob,Read" \
  --output-format json | jq '.result' > api-docs.json
```

---

## GitHub Actions Integration

```yaml
# .github/workflows/claude-review.yml
name: Claude Code Review

on:
  pull_request:
    types: [opened, synchronize]

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install Claude Code
        run: npm install -g @anthropic-ai/claude-code

      - name: Run Code Review
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          PR_DIFF=$(git diff origin/${{ github.base_ref }}...HEAD)
          REVIEW=$(echo "$PR_DIFF" | claude -p \
            --bare \
            --model sonnet \
            --max-turns 3 \
            --max-budget-usd 1.00 \
            --allowedTools "" \
            --output-format json \
            "Review this PR for bugs and security issues. Be brief.")

          echo "## Claude Code Review" > review.md
          echo "$REVIEW" | jq -r '.result' >> review.md
          gh pr comment ${{ github.event.pull_request.number }} --body-file review.md
```

---

## Error Handling in Scripts

```bash
#!/bin/bash

run_claude() {
  local PROMPT="$1"
  local MAX_RETRIES=3
  local RETRY_DELAY=5

  for i in $(seq 1 $MAX_RETRIES); do
    RESULT=$(claude -p "$PROMPT" \
      --allowedTools "Read,Bash" \
      --output-format json \
      --max-budget-usd 5.00 2>&1)

    EXIT_CODE=$?

    if [ $EXIT_CODE -eq 0 ]; then
      echo "$RESULT"
      return 0
    fi

    # Check if it's a retryable error
    if echo "$RESULT" | grep -q "rate_limit\|overloaded"; then
      echo "Retry $i/$MAX_RETRIES after ${RETRY_DELAY}s..." >&2
      sleep $RETRY_DELAY
      RETRY_DELAY=$((RETRY_DELAY * 2))  # Exponential backoff
    else
      echo "Non-retryable error: $RESULT" >&2
      return 1
    fi
  done

  echo "Max retries exceeded" >&2
  return 1
}

# Use it
run_claude "Fix the failing tests" || exit 1
```

---

## Structured Output with JSON Schema

Enforce that Claude returns a specific JSON structure:

```bash
SCHEMA='{
  "type": "object",
  "required": ["issues", "severity", "recommendation"],
  "properties": {
    "issues": {
      "type": "array",
      "items": { "type": "string" }
    },
    "severity": {
      "type": "string",
      "enum": ["critical", "high", "medium", "low"]
    },
    "recommendation": { "type": "string" }
  }
}'

claude -p "Analyze auth.ts for security issues" \
  --allowedTools "Read" \
  --output-format json \
  --json-schema "$SCHEMA" | \
  jq '.structured_output'
```

Output is guaranteed to match your schema or the call fails.

---

## Session Management in Scripts

```bash
# Create named session
SESSION=$(claude -p "Start analysis" \
  --name "auth-analysis-$(date +%Y%m%d)" \
  --output-format json | jq -r '.session_id')

# Continue it
claude -p "Go deeper on the JWT handling" \
  --resume "$SESSION" \
  --output-format json

# Fork it (new session ID, same history)
claude -p "Alternative approach" \
  --resume "$SESSION" \
  --fork-session \
  --output-format json
```

---

## Environment Variables for CI

```bash
# Authentication
export ANTHROPIC_API_KEY="sk-ant-..."       # Direct API key
export ANTHROPIC_AUTH_TOKEN="Bearer ..."     # OAuth/gateway token

# Model defaults
export CLAUDE_CODE_DEFAULT_MODEL="sonnet"

# Disable telemetry in CI
export CLAUDE_CODE_DISABLE_TELEMETRY=1

# Custom config location
export CLAUDE_CONFIG_DIR="/ci/claude-config"

# Memory disabled in CI (no persistence needed)
# Set via settings: "autoMemoryEnabled": false

# Override compaction threshold
export CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=90
```

---

## Summary

```
┌────────────────────────────────────────────────────────────────┐
│  HEADLESS/CI CHEATSHEET                                        │
├────────────────────────────────────────────────────────────────┤
│  Flag           : -p or --print (headless mode)                │
│  Formats        : text | json | stream-json                    │
│  Bare mode      : --bare (skips CLAUDE.md, hooks, MCP, etc.)   │
│  Auto-approve   : --allowedTools "Bash,Read,Edit"              │
│  Cost limit     : --max-budget-usd 5.00                        │
│  Turn limit     : --max-turns 10                               │
│  Session cont.  : --resume <session-id>                        │
│  Session fork   : --resume + --fork-session                    │
│  Schema enforce : --json-schema '{"type":"object",...}'        │
│  Piping         : cat file | claude -p "task"                  │
│  Retry on limit : use exponential backoff in shell script      │
│  Exit codes     : 0 = success, non-zero = error                │
└────────────────────────────────────────────────────────────────┘
```
