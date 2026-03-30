# Expert 09 — Building Your Own MCP Server

---

## Why Build a Custom MCP Server?

```
  Built-in tools:    Bash, Read, Edit, Write, Grep, Glob, WebFetch...
  MCP servers:       GitHub, Slack, Jira, Figma, Gmail...
  Custom MCP:        YOUR internal APIs, YOUR databases, YOUR tools

  Examples:
  • Company internal REST API → Claude can query it directly
  • PostgreSQL database → Claude can run read-only queries
  • Internal documentation search → Claude can find docs
  • Custom code analysis tool → Claude can run it
  • CI/CD pipeline API → Claude can trigger and monitor builds
```

---

## The MCP Protocol

MCP uses JSON-RPC 2.0 over stdin/stdout (for stdio servers).

```
  Claude Code                    Your MCP Server
  ──────────                     ───────────────

  [start server process]
                                 [process starts, awaits input]

  → {"jsonrpc":"2.0","id":1,
     "method":"initialize",
     "params":{"protocolVersion":"2024-11-05",...}}

                                 ← {"jsonrpc":"2.0","id":1,
                                    "result":{"protocolVersion":"...",
                                              "capabilities":{...}}}

  → {"jsonrpc":"2.0","id":2,
     "method":"tools/list"}

                                 ← {"jsonrpc":"2.0","id":2,
                                    "result":{"tools":[...]}}

  [Claude decides to use a tool]

  → {"jsonrpc":"2.0","id":3,
     "method":"tools/call",
     "params":{"name":"my_tool",
               "arguments":{...}}}

                                 ← {"jsonrpc":"2.0","id":3,
                                    "result":{"content":[...]}}
```

---

## Minimal Working Server (Node.js)

```javascript
// mcp-server/index.js
const readline = require('readline')

const rl = readline.createInterface({
  input: process.stdin,
  output: process.stdout,
  terminal: false
})

const tools = [
  {
    name: "get_user",
    description: "Get user details from our internal database",
    inputSchema: {
      type: "object",
      properties: {
        user_id: {
          type: "string",
          description: "The user's ID"
        }
      },
      required: ["user_id"]
    }
  },
  {
    name: "list_orders",
    description: "List recent orders for a user",
    inputSchema: {
      type: "object",
      properties: {
        user_id: { type: "string" },
        limit: { type: "integer", default: 10 }
      },
      required: ["user_id"]
    }
  }
]

async function handleRequest(request) {
  const { method, params, id } = request

  switch (method) {
    case "initialize":
      return {
        jsonrpc: "2.0", id,
        result: {
          protocolVersion: "2024-11-05",
          capabilities: { tools: {} },
          serverInfo: { name: "internal-api", version: "1.0.0" }
        }
      }

    case "tools/list":
      return {
        jsonrpc: "2.0", id,
        result: { tools }
      }

    case "tools/call": {
      const { name, arguments: args } = params

      try {
        let result

        if (name === "get_user") {
          // Call your internal API
          const response = await fetch(
            `https://api.internal.company.com/users/${args.user_id}`,
            { headers: { Authorization: `Bearer ${process.env.INTERNAL_API_KEY}` } }
          )
          result = await response.json()
        }
        else if (name === "list_orders") {
          const response = await fetch(
            `https://api.internal.company.com/orders?user=${args.user_id}&limit=${args.limit || 10}`,
            { headers: { Authorization: `Bearer ${process.env.INTERNAL_API_KEY}` } }
          )
          result = await response.json()
        }
        else {
          throw new Error(`Unknown tool: ${name}`)
        }

        return {
          jsonrpc: "2.0", id,
          result: {
            content: [{ type: "text", text: JSON.stringify(result, null, 2) }]
          }
        }
      } catch (error) {
        return {
          jsonrpc: "2.0", id,
          result: {
            content: [{ type: "text", text: `Error: ${error.message}` }],
            isError: true
          }
        }
      }
    }

    default:
      return {
        jsonrpc: "2.0", id,
        error: { code: -32601, message: `Method not found: ${method}` }
      }
  }
}

// Main loop
rl.on('line', async (line) => {
  try {
    const request = JSON.parse(line)
    const response = await handleRequest(request)
    process.stdout.write(JSON.stringify(response) + '\n')
  } catch (error) {
    process.stderr.write(`Error: ${error.message}\n`)
  }
})

process.stdin.on('end', () => process.exit(0))
```

---

## Using the Official SDK (Recommended)

```bash
npm install @modelcontextprotocol/sdk
```

```javascript
// mcp-server/server.js
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js"
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js"
import { z } from "zod"

const server = new McpServer({
  name: "internal-api",
  version: "1.0.0"
})

// Define tools with Zod schema validation
server.tool(
  "get_user",
  "Get user details from our internal database",
  {
    user_id: z.string().describe("The user's ID"),
    include_orders: z.boolean().optional().describe("Include order history")
  },
  async ({ user_id, include_orders }) => {
    const user = await fetchUser(user_id)

    if (include_orders) {
      user.orders = await fetchOrders(user_id)
    }

    return {
      content: [{ type: "text", text: JSON.stringify(user, null, 2) }]
    }
  }
)

server.tool(
  "search_docs",
  "Search internal documentation",
  {
    query: z.string().describe("Search query"),
    category: z.enum(["api", "guides", "runbooks"]).optional()
  },
  async ({ query, category }) => {
    const results = await searchDocumentation(query, category)
    return {
      content: [{ type: "text", text: results.map(r => r.title + ': ' + r.url).join('\n') }]
    }
  }
)

// Define resources (optional — for browsable data)
server.resource(
  "config://database",
  "Database configuration (read-only)",
  async () => ({
    contents: [{
      uri: "config://database",
      text: JSON.stringify({ host: process.env.DB_HOST, port: 5432 })
    }]
  })
)

// Start
const transport = new StdioServerTransport()
await server.connect(transport)
```

---

## Register with Claude Code

```json
// .mcp.json (project scope)
{
  "mcpServers": {
    "internal-api": {
      "type": "stdio",
      "command": "node",
      "args": ["./mcp-server/server.js"],
      "env": {
        "INTERNAL_API_KEY": "${INTERNAL_API_KEY}",
        "DB_HOST": "${DB_HOST}"
      }
    }
  }
}
```

```bash
# Or add via CLI
claude mcp add internal-api -- node ./mcp-server/server.js

# Test it
/mcp
# Should show: internal-api: connected
# Tools: mcp__internal-api__get_user, mcp__internal-api__search_docs
```

---

## Tool Result Content Types

Your tool can return different content types:

```javascript
// Text (most common)
return {
  content: [{ type: "text", text: "Result here" }]
}

// Multiple content blocks
return {
  content: [
    { type: "text", text: "Found user:" },
    { type: "text", text: JSON.stringify(user) }
  ]
}

// Error
return {
  content: [{ type: "text", text: "User not found" }],
  isError: true  // Claude knows it failed and can handle it
}

// Image (base64)
return {
  content: [{
    type: "image",
    data: imageBase64String,
    mimeType: "image/png"
  }]
}
```

---

## Python MCP Server

```python
# mcp-server/server.py
import asyncio
import json
import sys
from typing import Any

class MCPServer:
    def __init__(self):
        self.tools = [
            {
                "name": "query_database",
                "description": "Run a read-only SQL query",
                "inputSchema": {
                    "type": "object",
                    "properties": {
                        "sql": {"type": "string", "description": "SQL SELECT query"},
                        "limit": {"type": "integer", "default": 100}
                    },
                    "required": ["sql"]
                }
            }
        ]

    async def handle(self, request: dict) -> dict:
        method = request.get("method")
        id_ = request.get("id")

        if method == "initialize":
            return {"jsonrpc": "2.0", "id": id_, "result": {
                "protocolVersion": "2024-11-05",
                "capabilities": {"tools": {}},
                "serverInfo": {"name": "db-server", "version": "1.0.0"}
            }}

        elif method == "tools/list":
            return {"jsonrpc": "2.0", "id": id_,
                    "result": {"tools": self.tools}}

        elif method == "tools/call":
            tool_name = request["params"]["name"]
            args = request["params"]["arguments"]

            if tool_name == "query_database":
                try:
                    result = await self.run_query(args["sql"], args.get("limit", 100))
                    return {"jsonrpc": "2.0", "id": id_,
                            "result": {"content": [{"type": "text", "text": json.dumps(result)}]}}
                except Exception as e:
                    return {"jsonrpc": "2.0", "id": id_,
                            "result": {"content": [{"type": "text", "text": str(e)}], "isError": True}}

    async def run_query(self, sql: str, limit: int) -> list:
        # Your database logic here
        import asyncpg
        conn = await asyncpg.connect(os.environ["DATABASE_URL"])
        rows = await conn.fetch(f"{sql} LIMIT {limit}")
        return [dict(row) for row in rows]

    async def run(self):
        server = self
        reader = asyncio.StreamReader()
        await asyncio.get_event_loop().connect_read_pipe(
            lambda: asyncio.StreamReaderProtocol(reader), sys.stdin)

        async for line in reader:
            try:
                request = json.loads(line)
                response = await server.handle(request)
                sys.stdout.write(json.dumps(response) + "\n")
                sys.stdout.flush()
            except Exception as e:
                sys.stderr.write(f"Error: {e}\n")

asyncio.run(MCPServer().run())
```

---

## Best Practices

```
  ✓ Tools should be focused — one tool, one responsibility
  ✓ Descriptions must be specific — Claude uses them to decide when to call
  ✓ Return structured JSON when possible — Claude can parse it
  ✓ Always include isError: true for failures
  ✓ Log to stderr, never stdout (stdout is protocol transport)
  ✓ Handle graceful shutdown on EOF (stdin close)
  ✓ Keep startup time < 2 seconds
  ✓ Set timeout via MCP_TIMEOUT env var if slow

  ✗ Don't include credentials in tool descriptions
  ✗ Don't return huge responses (use pagination)
  ✗ Don't make irreversible changes without confirmation
  ✗ Don't write to stdout outside JSON-RPC responses
```

---

## Summary

```
┌────────────────────────────────────────────────────────────────┐
│  MCP SERVER DEV CHEATSHEET                                     │
├────────────────────────────────────────────────────────────────┤
│  Protocol       : JSON-RPC 2.0 over stdin/stdout               │
│  Methods        : initialize, tools/list, tools/call          │
│  SDK            : @modelcontextprotocol/sdk (npm)              │
│  Tool naming    : registered as mcp__<server>__<tool>          │
│  Error signal   : isError: true in result                      │
│  Content types  : text, image (base64)                         │
│  Register       : .mcp.json or claude mcp add                  │
│  Debug          : claude --debug mcp                           │
│  Test           : /mcp in session to check status              │
│  Log output     : stderr only (stdout is protocol)             │
└────────────────────────────────────────────────────────────────┘
```
