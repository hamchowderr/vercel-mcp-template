# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is an MCP (Model Context Protocol) server template designed to run on Vercel using serverless functions. It uses the `mcp-handler` package (v1.0.1) to create MCP servers that expose tools, prompts, and resources via HTTP/SSE transports.

## Development Commands

**Local development:**
```sh
vercel dev
```

**Test the MCP server (SSE transport):**
```sh
node scripts/test-client.mjs https://your-deployment-url.vercel.app
# Or for local testing:
node scripts/test-client.mjs http://localhost:3000
```

**Test the MCP server (Streamable HTTP transport):**
```sh
node scripts/test-streamable-http-client.mjs https://your-deployment-url.vercel.app
```

## Architecture

### Request Flow

```
Client Request
    ↓
Vercel Edge Network
    ↓
vercel.json rewrites all /(.+) → /api/server
    ↓
api/server.ts exports handler as GET/POST/DELETE
    ↓
mcp-handler routes by HTTP method + path
    ↓
- /mcp → Streamable HTTP transport
- /sse → SSE transport
    ↓
Zod validation → Handler execution → Response
```

### Core Components

**`api/server.ts`** - The main MCP server handler. All routes are rewritten to this file via `vercel.json`. This is where you define MCP tools, prompts, and resources.

**Pattern:**
```typescript
import { createMcpHandler } from "mcp-handler";
import { z } from "zod";

const handler = createMcpHandler((server) => {
  // Define tools, prompts, resources here
});

// CRITICAL: Export all three HTTP methods
export { handler as GET, handler as POST, handler as DELETE };
```

**Why all three HTTP methods?**
- `GET` - Used for SSE transport connections and listing operations
- `POST` - Primary method for tool invocations and MCP protocol requests
- `DELETE` - Used for cleanup/disconnection operations

**`mcp-handler`** - Wraps `@modelcontextprotocol/sdk` to create a Vercel-compatible serverless MCP server. Internally handles routing, transport negotiation, and protocol operations.

**`vercel.json`** - Rewrites ALL routes to `/api/server`, enabling the MCP server to handle both `/mcp` (HTTP transport) and `/sse` (SSE transport) endpoints from a single serverless function.

### Transport Types

The server supports two transport mechanisms:

**1. Streamable HTTP (`/mcp` endpoint)**
```javascript
import { StreamableHTTPClientTransport } from "@modelcontextprotocol/sdk/client/streamableHttp.js";
const transport = new StreamableHTTPClientTransport(new URL(`${origin}/mcp`));
```
- Request-response pattern with streaming support
- Better for serverless/stateless environments like Vercel
- Primary endpoint for MCP client integration

**2. SSE - Server-Sent Events (`/sse` endpoint)**
```javascript
import { SSEClientTransport } from "@modelcontextprotocol/sdk/client/sse.js";
const transport = new SSEClientTransport(new URL(`${origin}/sse`));
```
- Real-time, long-lived connections
- Server pushes events as a stream
- Uses HTTP GET for establishing connection

### Server API Methods

The `server` object passed to `createMcpHandler()` provides these methods:

**`server.tool(name, description, schema, handler)`**

Defines a callable tool with automatic Zod validation.

```typescript
server.tool(
  "get_weather",                      // Tool identifier
  "Get current weather at a location", // Human-readable description
  {                                   // Zod schema object
    latitude: z.number(),
    longitude: z.number(),
    city: z.string(),
  },
  async ({ latitude, longitude, city }) => {  // Typed handler
    // Implementation
    return {
      content: [{ type: "text", text: "..." }],
    };
  },
);
```

**Schema patterns:**
```typescript
// Simple validation
{ sides: z.number().int().min(2) }

// Complex validation
{
  email: z.string().email(),
  age: z.number().int().min(0).max(120),
  tags: z.array(z.string()).optional(),
  metadata: z.record(z.string(), z.any()).default({}),
}
```

**`server.prompt()` and `server.resource()`**

These capabilities exist in the MCP protocol but are not demonstrated in this template. Based on the client capabilities (see `scripts/test-client.mjs:16`), they follow similar patterns to `server.tool()`.

## Response Format

Tool handlers **must** return an object with a `content` array:

```typescript
return {
  content: [
    { type: "text", text: "Response message" },
  ],
};
```

**Multiple content items:**
```typescript
return {
  content: [
    { type: "text", text: "First piece of info" },
    { type: "text", text: "Second piece of info" },
  ],
};
```

**Content types:**
- `type: "text"` - Text content (requires `text` field)
- `type: "image"` - Image data (likely requires `url` or `data`)
- `type: "resource"` - Resource references (not demonstrated in template)

**Required fields:**
- `content` array (required)
- Each item must have `type`
- Type-specific fields (e.g., `text` for "text" type)

## Input Validation

Validation uses Zod schemas defined in the tool's third parameter. The `mcp-handler` automatically validates inputs before calling your handler.

**Common Zod validators:**
```typescript
z.string()           // Any string
z.number()           // Any number
z.number().int()     // Integer only
z.number().min(2)    // Minimum value
z.string().email()   // Email format
z.string().url()     // URL format
z.array(z.string())  // Array of strings
z.object({ ... })    // Nested object
.optional()          // Make field optional
.default(value)      // Default value
```

**Validation failures** are handled automatically by `mcp-handler` and return errors to the client before your handler is invoked.

## Error Handling

The template examples don't show explicit error handling, but recommended pattern:

```typescript
server.tool(
  "fetch_data",
  "Fetches data from external API",
  { url: z.string().url() },
  async ({ url }) => {
    try {
      const response = await fetch(url);
      if (!response.ok) {
        throw new Error(`HTTP ${response.status}: ${response.statusText}`);
      }
      const data = await response.json();
      return {
        content: [{ type: "text", text: JSON.stringify(data, null, 2) }],
      };
    } catch (error) {
      return {
        content: [{
          type: "text",
          text: `Error: ${error.message}`
        }],
      };
    }
  },
);
```

## MCP Client Integration

**For MCP client applications (like Claude Desktop):**
```
https://your-deployment-url.vercel.app/mcp
```

**Client connection pattern:**
```javascript
import { Client } from "@modelcontextprotocol/sdk/client/index.js";
import { StreamableHTTPClientTransport } from "@modelcontextprotocol/sdk/client/streamableHttp.js";

const transport = new StreamableHTTPClientTransport(
  new URL("https://your-deployment-url.vercel.app/mcp")
);

const client = new Client(
  { name: "my-client", version: "1.0.0" },
  { capabilities: { prompts: {}, resources: {}, tools: {} } }
);

await client.connect(transport);
const tools = await client.listTools();
```

## Vercel Configuration

**vercel.json structure:**
```json
{
  "rewrites": [{ "source": "/(.+)", "destination": "/api/server" }],
  "functions": {
    "api/server.ts": {
      "maxDuration": 60
    }
  }
}
```

**Critical settings:**
- **Fluid Compute**: Must be enabled in Vercel dashboard for efficient execution
- **Max Duration**:
  - Default: 60 seconds (all accounts)
  - Pro/Enterprise: Can increase to 800 seconds
- **Rewrites**: Routes ALL paths to the single serverless function

## Dependencies

**Production:**
- `mcp-handler` (^1.0.1) - Serverless MCP handler wrapping `@modelcontextprotocol/sdk`
- `zod` (^3.24.2) - Schema validation

**Development:**
- `@types/node` (^22.13.10) - TypeScript type definitions

**Package manager:** pnpm v9.4.0

## TypeScript Configuration

**Module system:** NodeNext (ES modules with Node.js resolution)
- `"type": "module"` in package.json
- Use `.mjs` for scripts or ES module syntax in `.ts` files

**Compiler options:**
- Target: ES2021
- Strict mode: enabled
- Declaration files: generated
- Includes: `src/**/*` and `api/**/*`

## Adding New Tools

1. Import dependencies in `api/server.ts`
2. Define tool inside `createMcpHandler()` callback
3. Use Zod schemas for type-safe parameters
4. Return `{ content: [...] }` format
5. Add error handling for external API calls

**Example:**
```typescript
server.tool(
  "my_tool",
  "Description of what the tool does",
  {
    param1: z.string(),
    param2: z.number().optional(),
  },
  async ({ param1, param2 }) => {
    try {
      // Tool implementation
      const result = await someOperation(param1, param2);
      return {
        content: [{ type: "text", text: result }],
      };
    } catch (error) {
      return {
        content: [{ type: "text", text: `Error: ${error.message}` }],
      };
    }
  },
);
```

## Extension Points

Based on MCP protocol capabilities, you can extend this template with:

1. **Multiple tools** - Add as many `server.tool()` calls as needed
2. **Prompts** - Use `server.prompt()` to define reusable prompt templates
3. **Resources** - Use `server.resource()` to expose data or documents
4. **External APIs** - Integrate with any HTTP API (see weather tool example)
5. **State management** - mcp-handler includes Redis support for caching/sessions

## Common Pitfalls

1. ❌ Forgetting to export all three HTTP methods (GET, POST, DELETE)
2. ❌ Returning raw strings instead of `{ content: [...] }` format
3. ❌ Not wrapping content in an array
4. ❌ Skipping Zod validation (loses type safety)
5. ❌ Exceeding maxDuration without upgrading Vercel plan
6. ❌ Not enabling Fluid Compute in Vercel settings
