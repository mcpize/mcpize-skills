# TypeScript MCP Server Patterns

These patterns come directly from the official mcpize TypeScript template. Follow them exactly so `mcpize dev` and `mcpize deploy` work out of the box.

## Server Setup (src/index.ts)

The entry point sets up Express + MCP SDK + dev logging. The template already scaffolds this — you mostly just need to replace the example tools with yours.

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StreamableHTTPServerTransport } from "@modelcontextprotocol/sdk/server/streamableHttp.js";
import express, { Request, Response } from "express";
import { z } from "zod";
import chalk from "chalk";
import { hello, echo } from "./tools.js";

// ============================================================================
// Dev Logging Utilities (auto-disabled in production)
// ============================================================================

const isDev = process.env.NODE_ENV !== "production";

function timestamp(): string {
  return new Date().toLocaleTimeString("en-US", { hour12: false });
}

function formatLatency(ms: number): string {
  if (ms < 100) return chalk.green(`${ms}ms`);
  if (ms < 500) return chalk.yellow(`${ms}ms`);
  return chalk.red(`${ms}ms`);
}

function truncate(str: string, maxLen = 60): string {
  if (str.length <= maxLen) return str;
  return str.slice(0, maxLen - 3) + "...";
}

function logRequest(method: string, params?: unknown): void {
  if (!isDev) return;
  const paramsStr = params ? chalk.gray(` ${truncate(JSON.stringify(params))}`) : "";
  console.log(`${chalk.gray(`[${timestamp()}]`)} ${chalk.cyan("→")} ${method}${paramsStr}`);
}

function logResponse(method: string, result: unknown, latencyMs: number): void {
  if (!isDev) return;
  const latency = formatLatency(latencyMs);
  if (method === "tools/call" && result) {
    const resultStr = typeof result === "string" ? result : JSON.stringify(result);
    console.log(
      `${chalk.gray(`[${timestamp()}]`)} ${chalk.green("←")} ${truncate(resultStr)} ${chalk.gray(`(${latency})`)}`
    );
  } else {
    console.log(`${chalk.gray(`[${timestamp()}]`)} ${chalk.green("✓")} ${method} ${chalk.gray(`(${latency})`)}`);
  }
}

function logError(method: string, error: unknown, latencyMs: number): void {
  const latency = formatLatency(latencyMs);
  let errorMsg: string;
  if (error instanceof Error) {
    errorMsg = error.message;
  } else if (typeof error === "object" && error !== null) {
    const rpcError = error as { message?: string; code?: number };
    errorMsg = rpcError.message || `Error ${rpcError.code || "unknown"}`;
  } else {
    errorMsg = String(error);
  }
  console.log(
    `${chalk.gray(`[${timestamp()}]`)} ${chalk.red("✖")} ${method} ${chalk.red(truncate(errorMsg))} ${chalk.gray(`(${latency})`)}`
  );
}

// ============================================================================
// MCP Server Setup
// ============================================================================

const server = new McpServer({
  name: "your-server-name",  // From brief's slug — auto-set by post-init
  version: "1.0.0",
});

// Register tools here (see Tool Registration below)

// ============================================================================
// Express App Setup
// ============================================================================

const app = express();
app.use(express.json());

// Health check — required for MCPize Cloud
app.get("/health", (_req: Request, res: Response) => {
  res.status(200).json({ status: "healthy" });
});

// MCP endpoint with dev logging and response capture
app.post("/mcp", async (req: Request, res: Response) => {
  const startTime = Date.now();
  const body = req.body;
  const method = body?.method || "unknown";
  const params = body?.params;

  // Log incoming request
  if (method === "tools/call") {
    const toolName = params?.name || "unknown";
    logRequest(`tools/call ${chalk.bold(toolName)}`, params?.arguments);
  } else if (method !== "notifications/initialized") {
    logRequest(method, params);
  }

  const transport = new StreamableHTTPServerTransport({
    sessionIdGenerator: undefined,
    enableJsonResponse: true,
  });

  // Capture response body for dev logging
  let responseBody = "";
  const originalWrite = res.write.bind(res) as typeof res.write;
  const originalEnd = res.end.bind(res) as typeof res.end;

  res.write = function (chunk: unknown, encodingOrCallback?: BufferEncoding | ((error: Error | null | undefined) => void), callback?: (error: Error | null | undefined) => void) {
    if (chunk) {
      responseBody += typeof chunk === "string" ? chunk : Buffer.from(chunk as ArrayBuffer).toString();
    }
    return originalWrite(chunk as string, encodingOrCallback as BufferEncoding, callback);
  };

  res.end = function (chunk?: unknown, encodingOrCallback?: BufferEncoding | (() => void), callback?: () => void) {
    if (chunk) {
      responseBody += typeof chunk === "string" ? chunk : Buffer.from(chunk as ArrayBuffer).toString();
    }
    // Log response
    if (method !== "notifications/initialized") {
      const latency = Date.now() - startTime;
      try {
        const rpcResponse = JSON.parse(responseBody) as { result?: unknown; error?: unknown };
        if (rpcResponse?.error) {
          logError(method, rpcResponse.error, latency);
        } else if (method === "tools/call") {
          const content = (rpcResponse?.result as { content?: Array<{ text?: string }> })?.content;
          const resultText = content?.[0]?.text;
          logResponse(method, resultText, latency);
        } else {
          logResponse(method, null, latency);
        }
      } catch {
        logResponse(method, null, latency);
      }
    }
    return originalEnd(chunk as string, encodingOrCallback as BufferEncoding, callback);
  };

  res.on("close", () => {
    transport.close();
  });

  await server.connect(transport);
  await transport.handleRequest(req, res, req.body);
});

// JSON error handler (Express defaults to HTML — we want JSON)
app.use((_err: unknown, _req: Request, res: Response, _next: Function) => {
  res.status(500).json({ error: "Internal server error" });
});

// Start server
const port = parseInt(process.env.PORT || "8080");
const httpServer = app.listen(port, () => {
  console.log();
  console.log(chalk.bold("MCP Server running on"), chalk.cyan(`http://localhost:${port}`));
  console.log(`  ${chalk.gray("Health:")} http://localhost:${port}/health`);
  console.log(`  ${chalk.gray("MCP:")}    http://localhost:${port}/mcp`);
  if (isDev) {
    console.log();
    console.log(chalk.gray("─".repeat(50)));
    console.log();
  }
});

// Graceful shutdown for Cloud Run
process.on("SIGTERM", () => {
  console.log("Received SIGTERM, shutting down...");
  httpServer.close(() => {
    process.exit(0);
  });
});
```

## Tool Registration

Every tool needs: name, metadata (title, description, schemas), and a handler function.

```typescript
server.registerTool(
  "tool_name",  // snake_case, matches brief's Core Tools table
  {
    title: "Human-Readable Title",
    description: "What this tool does — from the brief's Description column",
    inputSchema: {
      // Zod schemas — one per input parameter from the brief
      query: z.string().describe("What to search for"),
      limit: z.number().optional().default(10).describe("Max results to return"),
      chain: z.enum(["ethereum", "solana", "bsc"]).describe("Which blockchain"),
    },
    outputSchema: {
      // Zod schemas for the output shape
      results: z.array(z.object({
        name: z.string(),
        value: z.number(),
      })),
      total: z.number(),
    },
  },
  async ({ query, limit, chain }) => {
    // Your tool logic goes here
    const results = await fetchResults(query, chain, limit);

    const output = {
      results,
      total: results.length,
    };

    // ALWAYS return both content AND structuredContent
    return {
      content: [{ type: "text", text: JSON.stringify(output) }],
      structuredContent: output,
    };
  }
);
```

### Key things about tool registration

- **`inputSchema`** uses Zod — each field becomes a tool parameter. Always add `.describe()` so LLMs know what the parameter is for.
- **`outputSchema`** also uses Zod — defines the shape of `structuredContent`. LLMs and API consumers can rely on this shape.
- **Always return both `content` and `structuredContent`** — `content` is the text array that LLMs read, `structuredContent` is the typed object for programmatic use.
- **Tool names should be snake_case** — e.g., `get_whale_transactions`, not `getWhaleTransactions`.

## Error Handling in Tools

Wrap every tool handler in try/catch. Never let an error crash the server.

```typescript
async ({ query }) => {
  try {
    const data = await apiClient.fetch(query);

    const output = { results: data, cached: false };
    return {
      content: [{ type: "text", text: JSON.stringify(output) }],
      structuredContent: output,
    };
  } catch (error) {
    const message = error instanceof Error ? error.message : String(error);
    console.error(`[tool_name] Error:`, message);

    return {
      content: [{
        type: "text",
        text: JSON.stringify({
          error: message,
          suggestion: "Check that the API key is set correctly, or try again in a few seconds.",
        }),
      }],
      isError: true,
    };
  }
}
```

## API Client Wrapper

For servers that call external APIs (Type B from the brief), create a reusable client:

```typescript
// src/lib/api-client.ts
import NodeCache from "@cacheable/node-cache";

const cache = new NodeCache({ stdTTL: 300 }); // 5-minute default TTL

export class ApiClient {
  private baseUrl: string;
  private apiKey: string;

  constructor(baseUrl: string, apiKeyEnvVar: string) {
    this.baseUrl = baseUrl;
    const key = process.env[apiKeyEnvVar];
    if (!key) {
      console.warn(`Warning: ${apiKeyEnvVar} is not set`);
    }
    this.apiKey = key || "";
  }

  async get<T>(path: string, params?: Record<string, string>): Promise<T> {
    const url = new URL(path, this.baseUrl);
    if (params) {
      Object.entries(params).forEach(([k, v]) => url.searchParams.set(k, v));
    }

    // Check cache first
    const cacheKey = url.toString();
    const cached = cache.get<T>(cacheKey);
    if (cached) return cached;

    const response = await fetch(url.toString(), {
      headers: {
        "Authorization": `Bearer ${this.apiKey}`,
        "Content-Type": "application/json",
      },
    });

    if (!response.ok) {
      throw new Error(`API error: ${response.status} ${response.statusText}`);
    }

    const data = await response.json() as T;
    cache.set(cacheKey, data);
    return data;
  }
}
```

### Pagination

Always limit API response sizes. Some accounts/wallets have 10K+ records — fetching everything wastes API quota and takes 15+ seconds.

```typescript
async get<T>(path: string, params?: Record<string, string>, page = 1, limit = 100): Promise<T> {
  const url = new URL(path, this.baseUrl);
  if (params) {
    Object.entries(params).forEach(([k, v]) => url.searchParams.set(k, v));
  }
  // Always paginate — never fetch unbounded data
  url.searchParams.set("page", String(page));
  url.searchParams.set("offset", String(limit));
  // ...rest of method (cache check, fetch, etc.)
}
```

## Pure Functions in tools.ts

The template separates tool logic into `src/tools.ts` — pure functions with no MCP dependency. This makes them easy to unit test:

**Important**: Every result interface MUST include `[key: string]: unknown` — this index signature is required because `structuredContent` expects `Record<string, unknown>`. Without it, TypeScript will error on `structuredContent: output`.

```typescript
// src/tools.ts
export interface GetDataResult {
  [key: string]: unknown;  // Required for structuredContent compatibility
  results: string[];
  total: number;
}

export async function getData(query: string, limit = 10): Promise<GetDataResult> {
  const data = await apiClient.get("/search", { q: query, limit: String(limit) });
  return { results: data, total: data.length };
}
```

Then register in `src/index.ts`:

```typescript
import { getData } from "./tools.js";

server.registerTool("get_data", {
  title: "Get Data",
  description: "Search for data",
  inputSchema: { query: z.string().describe("Search query"), limit: z.number().optional().default(10) },
  outputSchema: { results: z.array(z.string()), total: z.number() },
}, async ({ query, limit }) => {
  const output = await getData(query, limit);
  return {
    content: [{ type: "text", text: JSON.stringify(output) }],
    structuredContent: output,
  };
});
```

### Splitting into multiple files

For servers with many tools, split `tools.ts` into multiple files:

```
src/
  index.ts          # Express + MCP server setup, tool registration
  tools/
    get-data.ts     # Pure function + types
    search.ts       # Another group
  lib/
    api-client.ts   # External API wrapper
```

## Required Dependencies

These are already in the template's `package.json`:

```json
{
  "dependencies": {
    "@modelcontextprotocol/sdk": "^1.23.0",
    "express": "^5.1.0",
    "zod": "^4.0.0",
    "chalk": "^5.4.1"
  },
  "devDependencies": {
    "@types/express": "^5.0.1",
    "@types/node": "^22.15.21",
    "tsx": "^4.19.4",
    "typescript": "^5.8.3",
    "vitest": "^3.0.0"
  }
}
```

Add these if your server needs them:
- `@cacheable/node-cache` — for caching API responses (drop-in replacement for abandoned `node-cache`)
- Other packages from the brief's Technical Stack

## Dev Logging

The template includes a comprehensive dev logging system with chalk-colored output:
- **Request logging** — shows method, tool name, arguments (truncated)
- **Response logging** — shows result text for tool calls, latency color-coded (green <100ms, yellow 100-500ms, red >500ms)
- **Error logging** — shows error messages with red indicator
- **Auto-disabled** in production (`NODE_ENV=production`)
- **Response body capture** — intercepts `res.write`/`res.end` to log actual tool results

Keep this code when editing `src/index.ts` — it's the block between `// Dev Logging Utilities` and `// MCP Server Setup`.
