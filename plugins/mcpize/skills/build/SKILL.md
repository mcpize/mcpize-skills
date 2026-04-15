---
name: build
description: "MCPize Builder — takes a brief/PRD (from /mcpize:idea or custom) and builds a production-ready MCP server, end to end. Part of the MCPize suite (mcpize.com). Parses brief to extract tools, data strategy, secrets, and technical stack. Scaffolds project via mcpize CLI with the right template, implements all MCP tools with proper input validation and error handling, configures mcpize.yaml with secrets/credentials and HTTP transport, writes tests that verify MCP protocol compliance, and generates README + CLAUDE.md. Use this skill whenever someone wants to build an MCP server from a brief or PRD, has a spec they want to turn into a working server, says 'build this MCP', wants to implement tools from a brief, or references mcpize-build. Also trigger on: build mcp, implement mcp server, code mcp from brief, scaffold mcp project, create mcp server from spec, mcpize build, turn brief into server, implement brief, build from prd, mcp server from scratch."
---

# MCPize: Build Your MCP Server

> Part of the **MCPize** suite (https://mcpize.com) — from idea to published MCP server.
>
> **Previous**: `/mcpize:idea` (discover & validate idea, produce brief)
> **This skill**: Build & Test the server
> **Next**: `/mcpize:publish` (publish & go-to-market)

Take a brief/PRD and turn it into a working, tested MCP server ready for deployment on MCPize.

Talk like a friendly California dev pair-programming with someone who's stoked to ship their first MCP server. Keep explanations clear, avoid jargon without context, celebrate wins ("Tests passing, nice!"), and stay calm on errors ("Let me dig into this..."). Professional but human — make building feel fun, not intimidating.

---

## The Job

0. **Parse brief & plan** — read the brief, understand what we're building, make a checklist
1. **Environment check** — make sure the tools are installed
2. **Scaffold project** — `mcpize init` with the right template
3. **Implement tools** — write all the MCP tools from the brief
4. **Error handling & caching** — make it production-solid
5. **Configuration** — mcpize.yaml, .env.example, .gitignore
6. **Documentation** — README.md, CLAUDE.md
7. **Testing** — unit tests + MCP protocol smoke test
8. **Verify & hand off** — make sure everything works, show next steps

---

## Step 0: Parse Brief & Plan

### Find the brief

Look for `mcp-brief-*.md` files in the current directory. If there are multiple, use `AskUserQuestion` to let the user pick:

- **header**: "Which brief should I build from?"
- **Options**: list each file name
- **multiSelect**: false

If no brief files exist, use `AskUserQuestion`:
- "Hey! I don't see any `mcp-brief-*.md` files here. Do you have a brief or PRD to work from?"
- **Options**: "Paste a file path", "I'll describe what I want to build", "Run /mcpize:idea first to create one"

A custom PRD/spec also works — it just needs to describe the tools the server should expose.

### Extract requirements

Read the brief and extract these key pieces:

| What | Where in brief | Example |
|------|---------------|---------|
| `name` / `slug` | Overview section | `whale-analytics-mcp` |
| `language` | Technical Stack | TypeScript or Python |
| `tools[]` | Core Tools table | name, description, input params, output shape |
| `mvpTools[]` | MVP Scope section | Which tools to build first |
| `dataStrategy` | Data Strategy section | Type A (pure computation) or Type B (needs external APIs) |
| `dataSources[]` | Data Strategy table | API names, auth methods, costs |
| `secrets[]` | Throughout brief | API keys the publisher provides (e.g., OPENAI_API_KEY) |
| `credentials[]` | If BYOK mentioned | API keys each subscriber brings (e.g., GITHUB_TOKEN) |
| `dependencies[]` | Technical Stack | npm/PyPI packages needed |
| `errorHandling[]` | Error & Edge Case table | What to do when things go wrong |
| `monetization` | Monetization section | Free / Freemium / Paid (informs `/mcpize:publish` pricing) |
| `dataAccessLevel` | Data Strategy section | Instant / Easy / Moderate / Hard / Barrier |

### Create the plan

Use `TodoWrite` to create a task list. Example plan:

- [ ] Check environment (mcpize CLI, Node.js or Python)
- [ ] Scaffold project via `mcpize init`
- [ ] Configure mcpize.yaml (secrets, transport)
- [ ] Implement shared utilities (API client, cache, error helpers)
- [ ] Implement tool: `get_xxx`
- [ ] Implement tool: `search_xxx`
- [ ] ... (one task per MVP tool)
- [ ] Create .env.example
- [ ] Write unit tests
- [ ] Write MCP protocol test script
- [ ] Generate README.md
- [ ] Generate CLAUDE.md
- [ ] Run end-to-end verification

### Confirm with user

Use `AskUserQuestion` to confirm before starting:
- **header**: "Ready to build"
- **text**: Show a summary of what you'll build — server name, language, tool list, secrets needed, data strategy
- **Options**: "Let's go!", "Change the scope", "Switch language"

---

## Step 1: Environment Check

Check that the required tools are installed:

```bash
# Is mcpize CLI installed?
which mcpize && mcpize --version

# Is the user logged in? (needed for deploy, secrets, and some init features)
mcpize whoami

# Is the runtime ready?
node --version   # For TypeScript projects
python3 --version && uv --version  # For Python projects
```

**If mcpize is not installed**, use `AskUserQuestion`:
- "Heads up — the `mcpize` CLI isn't installed yet. We need it to scaffold and deploy your server."
- **Options**: "Install it for me (`npm install -g mcpize`)", "I'll install it myself", "Skip it for now"

**If the brief lists external API keys** (`secrets[]` or `dataSources[]` with auth), warn early:
- "Heads up — this server needs API keys to work: `{list of keys}`. Make sure you have them before we start testing. I'll create a `.env.example` file during scaffolding so you know exactly what's needed."
- Don't block the build — scaffolding and coding work without keys. But testing will fail without them.

**If mcpize is installed but user is not logged in** (`mcpize whoami` fails), let the user know:
- "You'll need to log in before you can deploy or set secrets. You can do that now or later."
- Run `mcpize login` if they want to log in — it opens a browser window for auth.
- Building and testing locally works without login, so it's OK to defer this to Step 8.

### Pick the right template

| Brief says... | Template to use |
|---------------|----------------|
| TypeScript (or doesn't specify) | `typescript` |
| TypeScript + has OpenAPI spec | `typescript/openapi` |
| Python | `python` |
| Python + has OpenAPI spec | `python/openapi` |

Default to `typescript` — it's the most common and best-documented path.

---

## Step 2: Scaffold Project

### Initialize the project

```bash
mcpize init [server-name] --template [template]
cd [server-name]
```

The `mcpize init` command downloads the template, sets up the project structure, initializes git, and installs dependencies automatically. Use `--no-install` if you want to modify package.json first, then run `npm install` after.

If `mcpize` isn't available, you can manually create the project structure by following the template patterns from `${CLAUDE_SKILL_DIR}/references/typescript-patterns.md` or `${CLAUDE_SKILL_DIR}/references/python-patterns.md`.

### Clean up the template scaffolding

The template comes with example tools (`hello`, `echo`, etc.) — **remove them**. They're there to show the pattern, not to ship in your server. Replace them entirely with the tools from the brief.

### What the template gives you

The template already includes these files — you'll update them, not create from scratch:
- `README.md` — basic structure, update with your server's tools and description
- `CLAUDE.md` — dev guide, update with your project specifics
- `.env.example` — add your API key placeholders
- `.gitignore` — comprehensive, usually no changes needed
- `Dockerfile` — production-ready multi-stage build (Cloud Run ready)
- `Makefile` (Python only) — `make dev`, `make test`, `make lint`
- `tests/` — test scaffolding (TypeScript: `tests/tools.test.ts` with vitest; Python: `tests/test_tools.py` with pytest + `conftest.py`)

### Customize what got scaffolded

**TypeScript:**
1. Update `package.json`: set `description` from the brief, add any extra dependencies from the Technical Stack section
2. If the server calls external APIs, install caching: `npm install @cacheable/node-cache`

> **Dependency quality gate**: Before adding any npm/PyPI package, verify:
> - **Maintained**: last publish within the last 3 years
> - **Adopted**: 1K+ weekly npm downloads OR 500+ GitHub stars
> - If a package fails either check, find an actively maintained alternative.
3. Update `mcpize.yaml`: add `secrets` from the brief
4. **Preserve the dev logging code** in `src/index.ts` — it's the colorized request/response logger at the top of the file. Keep it, it's super useful during development and auto-disables in production.

**Python:**
1. Update `pyproject.toml`: set description, add dependencies (e.g., `httpx` for API calls, `cachetools` for caching)
2. Update `mcpize.yaml`: add `secrets` from the brief
3. Run `uv sync` to install new deps

> **Note**: The package directory and all references are automatically renamed by the post-init script during `mcpize init`. For example, `mcpize init whale-analytics` renames `src/my_mcp_server/` → `src/whale_analytics/` and updates all imports, pyproject.toml, mcpize.yaml, Dockerfile, Makefile, and tests automatically. You don't need to do this manually.

### Configure mcpize.yaml

Read `${CLAUDE_SKILL_DIR}/references/mcpize-yaml-reference.md` for the full schema. The key rules:

- **Always use `startCommand.type: http`** — this is the modern MCP transport (Streamable HTTP). Don't use stdio or sse for MCPize deployment.
- Add every secret from the brief under `secrets[]`
- If the brief mentions subscribers bringing their own API keys (BYOK), add `credentials[]` with `credentials_mode: per_user`
- **For OAuth-capable services** (GitHub, Google, Slack, Figma, etc.) — add `oauth_provider` and `oauth_scopes` to credentials. This gives users a "Connect with {Provider}" button instead of manual token entry. See the yaml reference for supported providers.
- Include `configSchema.source: code` so MCPize auto-discovers your tools after deploy

---

## Step 3: Implement Tools

This is where the real work happens. Before writing code, read the right reference file:
- **TypeScript**: read `${CLAUDE_SKILL_DIR}/references/typescript-patterns.md`
- **Python**: read `${CLAUDE_SKILL_DIR}/references/python-patterns.md`

These references contain the exact code patterns from the official mcpize templates — follow them so `mcpize dev` and `mcpize deploy` work out of the box.

### Verify APIs before coding

Before writing any API client code, actively research each external API from the brief:

1. **WebSearch** the current API docs — URLs, endpoints, and versions may have changed since the brief was written
2. **Check for breaking changes** — some APIs deprecate endpoints (e.g., Etherscan migrated from V1 to V2 in 2025, requiring a `chainid` parameter)
3. **Verify the free tier actually exists** — visit the pricing page. Some APIs claim "free tier" in docs but are actually paid-only or have removed free plans
4. **Hit every endpoint with a real `curl` call and save the responses** — don't just check that it returns 200. Save the full JSON response for each endpoint you'll use. Build your TypeScript interfaces and field-access paths from these real responses, not from API documentation. Docs are frequently outdated, incomplete, or describe a different API version than what you're actually hitting.

5. **Test with filters/combinations, not just basic queries** — many APIs return data for simple queries but return empty results when you add filters (e.g., `status=active`, `sort=date`). Test every parameter combination your tools will use. An API call that works without filters may silently fail with them due to query syntax issues.
6. **Watch out for URL encoding breaking API query syntax** — if the API uses special characters in query strings (`+` for OR, `()` for grouping, `:` for field matching), do NOT use `URLSearchParams` or similar built-in URL builders — they percent-encode these characters and break the query. Build the `search` parameter manually as a raw string in the URL.

This check prevents hours of debugging wrong response structures mid-build. API docs lie — real responses don't.

**To run these curl calls, you need the user's API keys.** If the server uses external APIs, stop and ask the user for their keys before writing the API client. List the keys needed, where to sign up, and the cost. Don't proceed with API client implementation until you have working keys — without them you can't verify response structures and will be coding blind.

### Build order

1. **Shared utilities first** — before you touch individual tools:
   - If the server calls external APIs (Type B from brief): create an API client wrapper with retry logic
   - Set up a caching layer (@cacheable/node-cache for TS, cachetools for Python)
   - Create an error formatting helper

2. **Then implement each MVP tool** from the brief's MVP Scope section:
   a. Define input schema from the Core Tools table's "Input" column
   b. Define output schema from the "Output" column
   c. Write the handler function — implement the actual logic
   d. Add error handling from the brief's Error & Edge Case table
   e. Test the endpoint with real data immediately — catch response structure mismatches one endpoint at a time, not all at the end
   f. Mark the TodoWrite task as done

### Code organization

**TypeScript — pure functions in tools.ts (same pattern as Python):**
```
src/
  index.ts          # Express + MCP server setup, tool registration
  tools.ts          # Pure functions — business logic only, no MCP dependency
  lib/
    api-client.ts   # External API wrapper (if server calls APIs)
    cache.ts        # Caching layer
```

**Python — tools as pure functions in a separate module:**
```
src/[package_name]/
  server.py         # FastMCP setup + tool registration
  tools.py          # Pure functions — all your tool logic lives here
  api_client.py     # External API wrapper (if needed)
```

### Rules that matter

1. **Validate all inputs** — use Zod schemas (TS) or type hints with docstrings (Python). Add `.describe()` on every field so LLMs know what each parameter is for.

2. **Return structured output** — In TypeScript, always return both `content` (text array for LLMs) AND `structuredContent` (typed object for programmatic use). In Python, just return a dict — FastMCP handles the rest.

3. **Never hardcode secrets** — always read from environment: `process.env.API_KEY` (TS) or `os.getenv("API_KEY")` (Python). The values get set via `mcpize secrets set` before deploy.

4. **Listen on the right port** — use `process.env.PORT || "8080"` (TS) or `int(os.getenv("PORT", "8080"))` (Python). MCPize Cloud sets this automatically.

5. **Include a health endpoint** — the template already has `GET /health` returning `{"status": "healthy"}`. Don't remove it — MCPize uses it for health checks.

6. **Handle SIGTERM** — Cloud Run sends SIGTERM before shutting down. The template handles this already.

---

## Step 4: Error Handling & Caching

### Error handling — the golden rules

The brief's Error & Edge Case table tells you exactly what to handle. On top of that:

- **Never crash** — always return a structured MCP response. For errors, set `isError: true` on the response.
- **Write error messages for LLMs** — the consumer of your tools is an AI, so error messages should explain what went wrong AND suggest what to try next.
- **Log with context** — include the API name, endpoint, HTTP status, and timing so debugging is easy.
- **Default every external field** — APIs often omit fields that are in their docs. Use `(field || 0)` for numbers, `(field || "")` for strings, `(field || [])` for arrays. One missing field silently cascades NaN/null through all downstream calculations.

See the code patterns in `${CLAUDE_SKILL_DIR}/references/typescript-patterns.md` or `${CLAUDE_SKILL_DIR}/references/python-patterns.md` — they show the exact try/catch structure to use.

### Caching

If the server calls external APIs, add caching. It reduces API costs, stays within rate limits, and makes responses faster:

- **TypeScript**: `@cacheable/node-cache` with TTLs (e.g., 60s for live prices, 3600s for metadata)
- **Python**: `cachetools.TTLCache`
- Make cache keys include all relevant parameters
- When serving cached data, include `cached: true` in the response so the LLM knows

---

## Step 5: Configuration

### mcpize.yaml

Read `${CLAUDE_SKILL_DIR}/references/mcpize-yaml-reference.md` for the full schema. Double-check:
- `startCommand.type: http` (always)
- All secrets from the brief are listed
- Credentials with `credentials_mode: per_user` if BYOK is needed
- `configSchema.source: code` for auto-discovery

### .env.example and .env

The template already has a `.env.example` with `PORT` and `NODE_ENV`/`ENV`. **Update it** to add the API keys from the brief:

```
# Server configuration
PORT=8080
NODE_ENV=development    # TypeScript
# ENV=development       # Python (set to "production" to disable dev logging)

# Required API keys — set these before running
# In production, use: mcpize secrets set KEY_NAME value
API_KEY=your-api-key-here
```

Then **also create a `.env` file** as a copy of `.env.example`. Tell the user to fill in their real API keys there for local development. Without this, `mcpize dev` will start the server but all API-dependent tools will fail.

`mcpize dev` automatically loads `.env` and `.env.local` files — no need for `dotenv` or `source .env`.

### .gitignore

The template includes a comprehensive `.gitignore` — verify it has `.env` and `.mcpize/` (it should). Usually no changes needed.

---

## Step 6: Documentation

The template already includes `README.md` and `CLAUDE.md` — update them with your project specifics rather than creating from scratch.

### README.md

```markdown
# [Server Name]

[One-liner from brief]

[![Available on MCPize](https://img.shields.io/badge/MCPize-Available-blue)](https://mcpize.com/servers/[slug])

## Tools

| Tool | Description |
|------|-------------|
| `tool_name` | What it does |

## Quick Start

### Install from MCPize (Recommended)
Visit [mcpize.com/servers/[slug]](https://mcpize.com/servers/[slug])

### Run Locally
[Language-specific install + run commands]

## Configuration

| Variable | Required | Description |
|----------|----------|-------------|
| `API_KEY` | Yes | Your API key for [service] |

## Development

\`\`\`bash
mcpize dev              # Start dev server with hot reload
mcpize dev --playground # Interactive testing in your browser
\`\`\`

## Deploy

\`\`\`bash
mcpize secrets set API_KEY your-key-here  # Set your secrets first
mcpize deploy                              # Ship it!
\`\`\`

## License
MIT
```

### CLAUDE.md

Give future Claude sessions context about the project:

```markdown
# [Server Name]

## Project Structure
[Describe directory layout — what's in src/, tools/, lib/]

## Adding a New Tool
[Step-by-step: create file, define schema, register, write test]

## Key Commands
- `mcpize dev` — start local dev server (port 3000, hot reload, auto-loads .env)
- `mcpize dev --playground` — test tools interactively via public tunnel + browser playground
- `mcpize dev --tunnel` — expose server via public URL (without opening playground)
- `mcpize doctor` — validate config, runtime, and Dockerfile before deploy
- `npm test` / `pytest` — run unit tests
- `bash test-mcp.sh` — MCP protocol smoke test
- `mcpize deploy` — deploy to MCPize Cloud

## Environment Variables
[List all env vars with descriptions]
```

---

## Step 7: Testing

Read `${CLAUDE_SKILL_DIR}/references/testing-patterns.md` for the full code patterns.

### Unit tests

Both templates ship with test scaffolding — update them with tests for your actual tools instead of the template's `hello`/`echo` examples:
- **TypeScript**: `tests/tools.test.ts` (vitest) — replace with tests for your tools
- **Python**: `tests/test_tools.py` + `tests/conftest.py` (pytest) — replace with tests for your tools

Test the pure logic — mock any external API calls.

### MCP protocol smoke test

Generate `test-mcp.sh` — a script that verifies the server actually speaks MCP correctly:

> **Important**: All curl calls to `/mcp` must include `-H "Accept: application/json, text/event-stream"`. The MCP SDK's Streamable HTTP transport validates this header and returns 406 Not Acceptable without it.

1. Start the server in the background
2. Wait for the health endpoint to respond
3. Send `initialize` handshake (JSON-RPC 2.0)
4. Call `tools/list` — check that all expected tools show up
5. Call `tools/call` for each tool with sample arguments from the brief
6. Send `ping` — verify health check works
7. Clean up — kill the server

This catches issues that unit tests miss: wrong JSON-RPC format, tools not registered, broken transport layer.

### Run everything

```bash
# Unit tests
npm test          # TypeScript
pytest            # Python

# Start server and run protocol tests
# Note: mcpize dev runs on port 3000 by default
mcpize dev &
sleep 3                                    # Give it a sec to start up
MCP_URL=http://localhost:3000 bash test-mcp.sh  # Run the smoke test
kill %1                                     # Stop the server
```

**CRITICAL**: Every curl call to `/mcp` in `test-mcp.sh` MUST include this header:
```bash
curl -X POST http://localhost:3000/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","method":"initialize","id":1,"params":{...}}'
```
Without `-H "Accept: application/json, text/event-stream"` the MCP SDK returns 406 Not Acceptable.

If all tests pass, the server is working correctly. If any fail, fix the issue and re-run.

### Live verification loop (acceptance criteria)

After unit tests and smoke tests pass, verify each tool works with **real data**. This is the most important step — don't skip it.

**Acceptance criteria** — every tool must pass ALL of these checks:

1. **Returns data** — response has `result` with non-empty, non-null content
2. **Structured JSON** — response data is a parsed JSON object with named fields, NOT a raw text dump. If a tool returns `"text": "{\"brand_name\":\"...\", ...}"` as a stringified blob, that's a fail — it should return structured fields that an LLM can reason about
3. **Meaningful content** — data actually contains useful information for the query. `total_found: 0` with an empty array on a common query (e.g., "ibuprofen recalls") means the tool is broken or the query parameters are wrong
4. **Multiple test cases** — test each tool with at least 2 different inputs, including one "obvious" query that MUST return results (e.g., "aspirin" for a drug search tool)

### Response quality checks (CRITICAL)

**Check 1: Structured vs raw text**

Tools should return parsed, structured JSON — not stringified text blobs. Compare:

Bad (raw text dump):
```json
{"type": "text", "text": "{\"brand_name\":\"Aspirin\",\"indications\":\"1 INDICATIONS AND USAGE...long text...\"}"}
```

Good (structured JSON):
```json
{"type": "text", "text": "{\"brand_name\":\"Aspirin\",\"generic_name\":\"ASPIRIN\",\"indications\":[\"Pain relief\",\"Fever reduction\"],\"warnings\":{\"contraindications\":[\"...\"],\"precautions\":[\"...\"]},\"dosage\":{\"adults\":\"325-650mg every 4-6 hours\",\"children\":\"See label\"}}"}
```

The difference: structured data has discrete fields that an LLM can reference individually. Raw text forces the LLM to re-parse unstructured content. When you parse API data, break it into logical fields — don't just dump the raw API response as-is.

**Check 2: Empty results on common queries**

If a tool returns empty results for a common/popular query, something is wrong:
- Wrong API endpoint or parameters
- Overly restrictive filters (e.g., filtering by status that excludes most results)
- Pagination issues (offset/limit wrong)
- API response structure changed

Fix: try the raw API call with curl first, see what it actually returns, then fix the tool's query logic.

### The loop

1. Start the server: `mcpize dev &` (or reuse if already running)
2. For each tool in the brief:
   a. Call it via MCP protocol (POST `/mcp` with `tools/call`, include `Accept: application/json, text/event-stream`)
   b. Run ALL 4 acceptance checks above
   c. Mark as PASS only if all 4 checks pass
3. If ANY tool fails:
   a. Read the error message and server logs
   b. Diagnose the root cause:
      - Raw text response? → Refactor tool to parse API data into structured fields
      - Empty results? → Test the raw API call with curl, compare parameters
      - Wrong structure? → Compare real API response vs your TypeScript interfaces
   c. Fix the code
   d. Re-run the failing tool — **go back to step 2**
4. Only move to Step 8 when **ALL tools pass ALL checks**

**Common failure patterns and fixes:**
- Raw text blob instead of JSON → parse API response into discrete fields with meaningful keys
- `NaN` in response → missing field from API, add defensive defaults: `(field || 0)`
- Empty array/null → API response structure changed, compare real response vs your interface. Also check: are you passing unnecessary filters that exclude results?
- Works without filters, empty with filters → URL encoding is breaking query syntax. `URLSearchParams` percent-encodes `+`, `(`, `)` which many APIs need as literal characters. Build the search/query param as a raw string instead
- 401/403 → API key not set or expired, check `.env`
- Timeout → API is slow, increase timeout or add pagination

> **Think like a 5x FAANG engineer**: don't ship until every tool returns structured, meaningful data for real queries. Unit tests prove your logic is correct. This loop proves your server actually works in the real world AND that the data quality is good enough for an LLM to use effectively.

---

## Step 8: Verification & Handoff

### Pre-deploy check

```bash
mcpize doctor  # Checks your config, runtime, and common gotchas
```

If `mcpize doctor` reports issues, fix them before moving on.

### Final summary

Present everything to the user in a clear, friendly way:

```
## Your server is ready! 🎉

**Server**: [name]
**Tools**: [count] implemented — [list them]
**Files**: [count] created

### Try it out locally
mcpize dev --playground

### API keys needed for local testing

Before you can test locally with real data, you'll need these API keys:

| Key | Where to get it | Cost |
|-----|----------------|------|
| `API_KEY_NAME` | [signup URL](link) | Free / $X/mo |

Do you have these keys? Paste them and I'll set up your `.env` file for local testing.

### Deploy to MCPize Cloud

# 1. Log in (if you haven't already — opens a browser)
mcpize login

# 2. Set your secrets
mcpize secrets set API_KEY your-key-here
[one line per secret from the brief]

# 3. Ship it!
mcpize deploy

### What's next
- Run `/mcpize:publish` to publish to the MCPize marketplace — handles SEO, pricing, logo, and listing in one command
- Or deploy manually with `mcpize deploy` and configure in the dashboard
- If deploy fails, run `mcpize diagnose` for AI-powered debugging
- Roll back a bad deploy with `mcpize rollback`
```

---

## Important Notes

- **HTTP transport always** — Streamable HTTP is the modern MCP standard. Never generate stdio or sse for MCPize deployment.
- **Brief is the source of truth** — tool names, descriptions, inputs, outputs all come from the brief. Don't invent tools that aren't in there.
- **MVP first** — implement only tools from the MVP Scope section, not the full Core Tools table (unless the user asks for everything).
- **Remove template example tools** — the scaffold comes with `hello` and `echo` tools. Delete them. Replace with the actual tools from the brief.
- **Preserve dev logging** — the template's logging middleware (the colorized request/response logger) is valuable. Keep it when editing `index.ts` or `server.py`.
- **Follow template patterns** — the generated code must match the official mcpize templates so `mcpize dev` and `mcpize deploy` work without surprises.
- **Don't skip testing** — the protocol smoke test catches real issues. Always run it before declaring the build done.
- **`mcpize dev --playground`** — this is the best way to test interactively. It creates a public tunnel (cloudflared/ngrok/localtunnel) and opens a browser-based playground where you can call tools and see responses. Always recommend it.
- **`mcpize dev` runs on port 3000 by default** — use `-p` flag to change it. The server code defaults to PORT 8080 (for production Cloud Run), but `mcpize dev` overrides it to 3000. Both are correct.
- **`mcpize dev` auto-loads `.env`** — it reads `.env` and `.env.local` files automatically. No need for `dotenv` packages or `source .env`.
- **`mcpize login` required for deploy** — building and testing locally works without auth, but `mcpize deploy`, `mcpize secrets`, and `mcpize status` all require login. Remind the user to run `mcpize login` before they try to deploy.
- **`.env` for local dev** — if the server needs API keys, create a `.env` file with the keys so local testing works. The `.env` file should be in `.gitignore`.
- **`mcpize diagnose`** — if deployment fails, this opens AI-powered diagnosis in the browser. Mention it in the handoff.
- **Request API keys early** — if the server uses external APIs, stop and ask the user for their keys in Step 3 (before writing the API client). List what's needed, where to sign up, and the cost. Wait for the user to provide the keys before proceeding — without them you can't verify real API response structures and will be coding blind.
