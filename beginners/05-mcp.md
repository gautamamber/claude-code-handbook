# 05 — MCP: Model Context Protocol

---

## What Is MCP?

MCP (Model Context Protocol) is a standard for connecting Claude to external tools and services.
Instead of Claude Code having built-in GitHub/Slack/Jira integrations,
MCP servers act as bridges — each running as a separate process or HTTP endpoint.

```
  ┌─────────────────────────────────────────────────────────────────┐
  │                     Claude Code Session                         │
  │                                                                 │
  │   Claude  ──►  mcp__github__search_repositories               │
  │                           │                                     │
  └───────────────────────────┼─────────────────────────────────────┘
                              │  MCP protocol
                              ▼
                ┌─────────────────────────┐
                │   GitHub MCP Server     │  (separate process)
                │   npx @github/mcp       │
                └────────────┬────────────┘
                             │
                             ▼
                       GitHub API
```

---

## The Deferred Loading Insight

This is the most important thing to understand about MCP.

```
  WITHOUT deferred loading (naive approach):
  ┌────────────────────────────────────────────────────────┐
  │  Session start                                         │
  │  Load all MCP tool definitions into context           │
  │  20 MCP servers × 10 tools × 500 tokens each          │
  │  = 100,000 tokens consumed before you type anything   │
  └────────────────────────────────────────────────────────┘

  WITH deferred loading (how Claude Code actually works):
  ┌────────────────────────────────────────────────────────┐
  │  Session start                                         │
  │  Only tool NAMES are in context (~5 tokens each)      │
  │  20 MCP servers × 10 tools × 5 tokens = 1,000 tokens │
  │                                                        │
  │  When Claude needs a tool:                             │
  │  → tool_search call loads that tool's full definition  │
  │  → Tool becomes usable in that moment                  │
  └────────────────────────────────────────────────────────┘
```

This is why MCP tools sometimes "appear" mid-conversation — they weren't loaded yet.

---

## Configuration Scopes

```
  ~/.claude/.mcp.json          ← user scope (applies to all projects)
  .mcp.json                    ← project scope (commit to git, shared)
```

Same precedence rules as settings.json — project overrides user for same keys.

---

## Server Types

```
  ┌──────────────────────────────────────────────────────────────┐
  │  TYPE: stdio                                                 │
  │                                                              │
  │  Runs a local process. Communication via stdin/stdout.       │
  │  Most common type. Works offline.                           │
  │                                                              │
  │  "type": "stdio",                                           │
  │  "command": "npx",                                          │
  │  "args": ["@github/mcp-server"]                             │
  └──────────────────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────────────┐
  │  TYPE: http                                                  │
  │                                                              │
  │  Remote server at a fixed URL.                               │
  │  REST-style. One request per tool call.                     │
  │                                                              │
  │  "type": "http",                                            │
  │  "url": "http://localhost:8080/mcp"                         │
  └──────────────────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────────────┐
  │  TYPE: sse  (Server-Sent Events)                             │
  │                                                              │
  │  Streaming HTTP connection. Server pushes updates.          │
  │                                                              │
  │  "type": "sse",                                             │
  │  "url": "http://localhost:8080/sse"                         │
  └──────────────────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────────────┐
  │  TYPE: ws  (WebSocket)                                       │
  │                                                              │
  │  Bidirectional streaming. Low latency.                      │
  │                                                              │
  │  "type": "ws",                                              │
  │  "url": "ws://localhost:8080/mcp"                           │
  └──────────────────────────────────────────────────────────────┘
```

---

## .mcp.json Configuration

```json
{
  "mcpServers": {

    "github": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@github/mcp-server"],
      "env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}"
      }
    },

    "slack": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@slack/mcp-server"],
      "env": {
        "SLACK_BOT_TOKEN": "${SLACK_BOT_TOKEN}"
      }
    },

    "my-internal-api": {
      "type": "http",
      "url": "http://internal-tools.company.com/mcp",
      "headers": {
        "Authorization": "Bearer ${INTERNAL_API_KEY}"
      }
    },

    "local-scripts": {
      "type": "stdio",
      "command": "node",
      "args": ["${CLAUDE_PROJECT_DIR}/mcp-server/index.js"]
    }

  }
}
```

---

## Environment Variable Expansion

Inside `.mcp.json`, you can reference environment variables:

```
${VARIABLE_NAME}           → expanded from shell environment
${CLAUDE_PROJECT_DIR}      → absolute path to project root
${HOME}                    → user home directory
```

```json
{
  "mcpServers": {
    "my-server": {
      "command": "node",
      "args": ["${CLAUDE_PROJECT_DIR}/mcp/server.js"],
      "env": {
        "DB_URL": "${DATABASE_URL}",
        "API_KEY": "${MY_API_KEY}"
      }
    }
  }
}
```

---

## Tool Naming Convention

Every MCP tool follows a predictable naming pattern:

```
  mcp__<server_name>__<tool_name>

  Examples:
    mcp__github__search_repositories
    mcp__github__create_pull_request
    mcp__slack__send_message
    mcp__slack__search_channels
    mcp__atlassian__getJiraIssue
    mcp__atlassian__createConfluencePage
```

This naming matters for:
- Hook matchers
- Permission allow/deny rules
- Referring to tools in conversations

---

## Hook Matchers for MCP Tools

```
  "matcher": "mcp__.*"              ← ALL MCP tools from ALL servers
  "matcher": "mcp__github__.*"      ← ALL tools from GitHub server
  "matcher": "mcp__github__create_.*"  ← only GitHub create* tools
  "matcher": "mcp__slack__send_message"  ← exact tool match
```

---

## How a Tool Call Flows

```
  Claude decides to search GitHub issues
           │
           ▼
  tool_search("github issues")       ← deferred load: get full tool definition
           │
           ▼
  Definition loaded:
    name: mcp__github__search_issues
    params: { query, repo, state, ... }
           │
           ▼
  PreToolUse hook fires (if configured)
           │
           ▼
  Permission check
           │
           ▼
  MCP call sent to GitHub server process
  ┌──────────────────────────────┐
  │  JSON-RPC over stdio:        │
  │  {                           │
  │    "method": "tools/call",   │
  │    "params": {               │
  │      "name": "search_issues",│
  │      "arguments": { ... }    │
  │    }                         │
  │  }                           │
  └──────────────────────────────┘
           │
           ▼
  MCP server responds
           │
           ▼
  PostToolUse hook fires
           │
           ▼
  Result returned to Claude
```

---

## Dynamic Updates

Changes to `.mcp.json` are picked up automatically via file watcher.
No restart needed. New servers start on next tool access.

```
  Edit .mcp.json → add new server
          │
          ▼
  ConfigChange hook fires
          │
          ▼
  New server available in current session
```

---

## MCP Prompts (Reusable Prompt Templates)

MCP servers can define prompts — reusable templates invoked as slash commands:

```
  /mcp-prompt-name [arguments]

  Example:
  /create-pr --title "fix auth" --branch "feature/auth"
```

Results are returned as structured JSON and injected into Claude's context.

---

## Elicitation — MCP Requesting User Input

Some MCP tools need user input mid-execution.

```
  MCP tool running...
           │
           ▼
  Server sends elicitation request:
  "What environment should I deploy to? (staging/prod)"
           │
           ▼
  Elicitation hook fires
           │
           ▼
  Claude prompts user (or hook auto-responds)
           │
           ▼
  ElicitationResult hook fires
           │
           ▼
  MCP tool continues with user's answer
```

---

## Output Limits

MCP tool responses have size limits. Claude Code shows a warning if exceeded
but does NOT block the tool.

```
  Tool returns 50,000 tokens of output
           │
           ▼
  Warning: "MCP tool output exceeded limit"
           │
           ▼
  Tool result still returned (truncated or partial)
  Execution continues
```

---

## Common MCP Servers

```
  @github/mcp-server          GitHub issues, PRs, repos, code search
  @slack/mcp-server           Slack messages, channels, search
  @atlassian/mcp-server       Jira, Confluence
  @figma/mcp-server           Figma designs, components
  @gmail/mcp-server           Gmail search, drafts
  @google-calendar/mcp        Calendar events, scheduling
```

---

## Building Your Own MCP Server (Concept)

```
  Any process that:
  1. Reads JSON-RPC from stdin
  2. Writes JSON-RPC responses to stdout
  3. Declares tools with name + description + parameter schema

  Minimal stdio server (Node.js):

  process.stdin.on('data', (data) => {
    const request = JSON.parse(data)

    if (request.method === 'tools/list') {
      process.stdout.write(JSON.stringify({
        tools: [
          {
            name: 'my_tool',
            description: 'Does something useful',
            inputSchema: { type: 'object', properties: { ... } }
          }
        ]
      }))
    }

    if (request.method === 'tools/call') {
      // execute tool
      // write result to stdout
    }
  })
```

---

## Summary

```
┌────────────────────────────────────────────────────────────────┐
│  MCP CHEATSHEET                                                │
├────────────────────────────────────────────────────────────────┤
│  Tool naming    : mcp__<server>__<tool>                        │
│  Config files   : .mcp.json (project), ~/.claude/.mcp.json    │
│  Server types   : stdio | http | sse | ws                      │
│  Loading        : DEFERRED — names at startup, defs on demand  │
│  Dynamic reload : .mcp.json changes apply without restart      │
│  Hook matchers  : mcp__.* for all, mcp__github__.* for one     │
│  Env expansion  : ${VAR}, ${CLAUDE_PROJECT_DIR}                │
│  Prompts        : /mcp-prompt-name as slash commands           │
└────────────────────────────────────────────────────────────────┘
```
