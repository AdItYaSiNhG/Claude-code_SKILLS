---
name: integrations-mcp
description: MCP (Model Context Protocol) server configuration, management, and custom server development. Use to connect Claude Code to Slack, Notion, Jira, Figma, Datadog, GitHub, databases, or any internal tool via MCP.
---

The user wants to connect Claude Code to an external tool, or build a custom MCP server. Apply the relevant section.

## MCP Configuration (Claude Code)

### Adding a pre-built MCP server
```json
// ~/.claude/claude_desktop_config.json  (Claude Desktop)
// .claude/claude.json  (per-project, checked into repo)
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "${GITHUB_TOKEN}"
      }
    },
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres", "${DATABASE_URL}"]
    },
    "slack": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-slack"],
      "env": {
        "SLACK_BOT_TOKEN": "${SLACK_BOT_TOKEN}",
        "SLACK_TEAM_ID": "${SLACK_TEAM_ID}"
      }
    },
    "figma": {
      "url": "https://mcp.figma.com/mcp",
      "type": "url"
    }
  }
}
```

### Available pre-built servers (official + community)
| Tool | Package / URL |
|---|---|
| GitHub | `@modelcontextprotocol/server-github` |
| PostgreSQL | `@modelcontextprotocol/server-postgres` |
| Slack | `@modelcontextprotocol/server-slack` |
| Figma | `https://mcp.figma.com/mcp` (URL type) |
| Vercel | `https://mcp.vercel.com` (URL type) |
| Postman | `https://mcp.postman.com/minimal` (URL type) |
| Jira / Confluence | `@modelcontextprotocol/server-atlassian` |
| Notion | `@modelcontextprotocol/server-notion` |
| Datadog | Community: `mcp-datadog` |
| AWS | Community: `aws-mcp-server` |
| Filesystem | `@modelcontextprotocol/server-filesystem` |
| Fetch / HTTP | `@modelcontextprotocol/server-fetch` |
| SQLite | `@modelcontextprotocol/server-sqlite` |

## Building a Custom MCP Server

### Minimal TypeScript MCP server
```ts
// src/server.ts
import { Server } from "@modelcontextprotocol/sdk/server/index.js"
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js"
import {
  CallToolRequestSchema,
  ListToolsRequestSchema,
} from "@modelcontextprotocol/sdk/types.js"

const server = new Server(
  { name: "orders-mcp", version: "1.0.0" },
  { capabilities: { tools: {} } }
)

// Register available tools
server.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools: [
    {
      name: "get_order",
      description: "Fetch an order by ID from the orders service. Returns order details including status, items, and total.",
      inputSchema: {
        type: "object",
        properties: {
          order_id: { type: "string", description: "UUID of the order" },
        },
        required: ["order_id"],
      },
    },
    {
      name: "list_orders",
      description: "List orders for a customer. Use when the user asks about recent orders.",
      inputSchema: {
        type: "object",
        properties: {
          customer_email: { type: "string" },
          limit: { type: "number", default: 10, maximum: 50 },
          status: { type: "string", enum: ["pending", "confirmed", "shipped", "cancelled"] },
        },
        required: ["customer_email"],
      },
    },
  ],
}))

// Handle tool calls
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const { name, arguments: args } = request.params

  switch (name) {
    case "get_order": {
      const order = await fetchOrder(args.order_id as string)
      return {
        content: [{ type: "text", text: JSON.stringify(order, null, 2) }],
      }
    }
    case "list_orders": {
      const orders = await listOrders(args as { customer_email: string; limit?: number })
      return {
        content: [{ type: "text", text: JSON.stringify(orders, null, 2) }],
      }
    }
    default:
      throw new Error(`Unknown tool: ${name}`)
  }
})

// Start server
const transport = new StdioServerTransport()
await server.connect(transport)
```

### Package.json for MCP server
```json
{
  "name": "orders-mcp-server",
  "version": "1.0.0",
  "type": "module",
  "bin": { "orders-mcp": "./dist/server.js" },
  "scripts": {
    "build": "tsc",
    "dev": "tsx src/server.ts",
    "start": "node dist/server.js"
  },
  "dependencies": {
    "@modelcontextprotocol/sdk": "^1.0.0"
  },
  "devDependencies": {
    "typescript": "^5.0.0",
    "tsx": "^4.0.0"
  }
}
```

### Register the custom server
```json
// .claude/claude.json
{
  "mcpServers": {
    "orders": {
      "command": "node",
      "args": ["/path/to/orders-mcp-server/dist/server.js"],
      "env": {
        "ORDERS_API_URL": "https://api.internal/v1",
        "ORDERS_API_KEY": "${ORDERS_API_KEY}"
      }
    }
  }
}
```

## MCP Server Best Practices

### Tool design for Claude
```
1. Tool description = the most important thing
   Bad:  "Get order"
   Good: "Fetch a single order by UUID from the orders service.
          Returns status, line items, totals, and shipping address.
          Use when the user provides a specific order ID."

2. Return structured data, not prose
   Bad:  "Order 123 is confirmed with total $45.00"
   Good: {"id": "123", "status": "confirmed", "total_cents": 4500, ...}
   → Let Claude format the response for the user

3. Handle errors gracefully — return them to Claude, not exceptions
   return {
     content: [{ type: "text", text: JSON.stringify({error: "NOT_FOUND", id: args.order_id}) }],
     isError: true,
   }

4. Keep tools atomic — one action per tool
   Bad:  "search_and_update_order"
   Good: separate "search_orders" + "update_order_status"

5. Include examples in descriptions for complex inputs
   "customer_email: customer's email address, e.g. user@example.com"
```

### Authentication patterns
```ts
// OAuth2 (for user-delegated tools)
const token = await getOAuth2Token({
  clientId: process.env.CLIENT_ID,
  clientSecret: process.env.CLIENT_SECRET,
  scope: "orders:read orders:write",
})

// API Key (for service-to-service)
const headers = { Authorization: `Bearer ${process.env.API_KEY}` }

// OIDC (for internal services with IAM)
const idToken = await getOIDCToken(process.env.SERVICE_ACCOUNT)
```

## Output Format

1. For configuring an existing server: show the exact JSON block to add to `claude.json`
2. For building a custom server: show the full `server.ts` with all tool schemas
3. Always include the registration config alongside the server code
4. Flag any auth requirements and how to supply credentials safely (env vars, not hardcoded)
