# Testing Patterns for MCP Servers

Two kinds of tests: unit tests for tool logic, and a protocol smoke test to verify the server speaks MCP correctly.

> **Python note**: The template already ships with `tests/test_tools.py` and `tests/conftest.py` — update them with tests for your actual tools instead of creating from scratch.

## MCP Protocol Smoke Test (test-mcp.sh)

This is the most important test — it verifies your server actually works as an MCP server. Generate this for every project.

> **Important**: All curl calls to `/mcp` must include `-H "Accept: application/json, text/event-stream"`. The MCP SDK's Streamable HTTP transport validates this header and returns 406 Not Acceptable without it.

```bash
#!/bin/bash
# MCP Protocol Smoke Test
# Usage: Start your server first, then run: bash test-mcp.sh

BASE_URL="${MCP_URL:-http://localhost:3000}"
MCP_ENDPOINT="$BASE_URL/mcp"
HEALTH_ENDPOINT="$BASE_URL/health"
PASSED=0
FAILED=0

# Colors
GREEN='\033[0;32m'
RED='\033[0;31m'
NC='\033[0m'

pass() { echo -e "${GREEN}PASS${NC} $1"; PASSED=$((PASSED + 1)); }
fail() { echo -e "${RED}FAIL${NC} $1: $2"; FAILED=$((FAILED + 1)); }

echo "Testing MCP server at $BASE_URL"
echo "================================"

# 1. Health check
echo ""
echo "--- Health Check ---"
HEALTH=$(curl -sf "$HEALTH_ENDPOINT" 2>/dev/null) || true
if echo "$HEALTH" | grep -q "healthy"; then
  pass "GET /health returns healthy"
else
  fail "GET /health" "Expected 'healthy' in response, got: $HEALTH"
fi

# 2. Initialize handshake
echo ""
echo "--- MCP Initialize ---"
INIT_RESPONSE=$(curl -sf -X POST "$MCP_ENDPOINT" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "initialize",
    "params": {
      "protocolVersion": "2025-03-26",
      "capabilities": {},
      "clientInfo": { "name": "smoke-test", "version": "1.0" }
    }
  }' 2>/dev/null) || true

if echo "$INIT_RESPONSE" | grep -q '"result"'; then
  pass "initialize returns result"
else
  fail "initialize" "No 'result' in response: $INIT_RESPONSE"
fi

# 3. List tools
echo ""
echo "--- List Tools ---"
TOOLS_RESPONSE=$(curl -sf -X POST "$MCP_ENDPOINT" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{
    "jsonrpc": "2.0",
    "id": 2,
    "method": "tools/list",
    "params": {}
  }' 2>/dev/null) || true

if echo "$TOOLS_RESPONSE" | grep -q '"tools"'; then
  pass "tools/list returns tools array"
  # Count tools
  TOOL_COUNT=$(echo "$TOOLS_RESPONSE" | python3 -c "import sys,json; print(len(json.load(sys.stdin)['result']['tools']))" 2>/dev/null || echo "?")
  echo "     Found $TOOL_COUNT tool(s)"
else
  fail "tools/list" "No 'tools' in response: $TOOLS_RESPONSE"
fi

# 4. Check specific tools exist
# CUSTOMIZE: Replace these with your actual tool names from the brief
EXPECTED_TOOLS=("hello" "echo")
for TOOL in "${EXPECTED_TOOLS[@]}"; do
  if echo "$TOOLS_RESPONSE" | grep -q "\"$TOOL\""; then
    pass "Tool '$TOOL' is registered"
  else
    fail "Tool '$TOOL'" "Not found in tools/list response"
  fi
done

# 5. Call a tool
# CUSTOMIZE: Replace with a real tool call from your brief
echo ""
echo "--- Call Tool ---"
CALL_RESPONSE=$(curl -sf -X POST "$MCP_ENDPOINT" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{
    "jsonrpc": "2.0",
    "id": 3,
    "method": "tools/call",
    "params": {
      "name": "hello",
      "arguments": { "name": "Test" }
    }
  }' 2>/dev/null) || true

if echo "$CALL_RESPONSE" | grep -q '"content"'; then
  pass "tools/call returns content"
else
  fail "tools/call" "No 'content' in response: $CALL_RESPONSE"
fi

# 6. Ping (MCP health check)
echo ""
echo "--- Ping ---"
PING_RESPONSE=$(curl -sf -X POST "$MCP_ENDPOINT" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{
    "jsonrpc": "2.0",
    "id": 4,
    "method": "ping",
    "params": {}
  }' 2>/dev/null) || true

if echo "$PING_RESPONSE" | grep -q '"result"'; then
  pass "ping returns result"
else
  fail "ping" "No 'result' in response: $PING_RESPONSE"
fi

# Summary
echo ""
echo "================================"
echo -e "Results: ${GREEN}$PASSED passed${NC}, ${RED}$FAILED failed${NC}"

if [ $FAILED -gt 0 ]; then
  exit 1
fi
```

### How to customize the test

Replace these parts for each project:
1. **`EXPECTED_TOOLS`** — list all tool names from the brief's MVP Scope
2. **Tool call example** — change the `tools/call` section to call one of your actual tools with realistic arguments from the brief
3. **Add more tool calls** — copy the `tools/call` block for each MVP tool

### Running the test

```bash
# Start the server first
npm run dev &   # or: mcpize dev &
sleep 3         # Wait for it to start

# Run the test
bash test-mcp.sh

# Clean up
kill %1
```

Or if the server is already running:
```bash
MCP_URL=http://localhost:3000 bash test-mcp.sh
```

## TypeScript Unit Tests (vitest)

The template already includes `vitest` in devDependencies and `"test": "vitest run"` in package.json scripts. The template's `tests/tools.test.ts` has example tests — replace them with tests for your actual tools.

### Testing pure tool logic

```typescript
// tests/tools.test.ts
import { describe, it, expect } from "vitest";

// Import the pure function from tools.ts, not the MCP handler
import { calculateScore } from "../src/tools.js";

describe("calculateScore", () => {
  it("returns a score between 0 and 100", () => {
    const result = calculateScore({ value: 50, max: 100 });
    expect(result.score).toBeGreaterThanOrEqual(0);
    expect(result.score).toBeLessThanOrEqual(100);
  });

  it("handles edge case: zero value", () => {
    const result = calculateScore({ value: 0, max: 100 });
    expect(result.score).toBe(0);
  });

  it("handles edge case: max value", () => {
    const result = calculateScore({ value: 100, max: 100 });
    expect(result.score).toBe(100);
  });
});
```

> **ethers.js v6 note**: If your server uses `ethers.isAddress()` for Ethereum address validation, be aware that v6 validates EIP-55 checksums on mixed-case addresses. In tests, use all-lowercase addresses (e.g., `"0x742d35cc6634c0532925a3b844bc9e7595f2bd59"`) or get the correct checksum via `getAddress("0x...")`. Mixed-case with wrong checksums will silently fail validation.

### Cache isolation in tests

If your tools use caching (`@cacheable/node-cache` / `cachetools`), cache state persists between test runs. Mock the cache to prevent cross-test contamination:

**TypeScript (vitest):**
```typescript
// Mock cache — prevents tests from reading each other's cached results
vi.mock("../src/lib/cache.js", async () => {
  const actual = await vi.importActual<typeof import("../src/lib/cache.js")>("../src/lib/cache.js");
  return {
    ...actual,
    getCached: vi.fn().mockReturnValue(undefined),
    setCached: vi.fn(),
  };
});
```

**Python (pytest):** Use a fresh cache instance per test via a fixture in `conftest.py`, or mock with `unittest.mock.patch`.

### Mocking API calls

```typescript
import { describe, it, expect, vi, beforeEach } from "vitest";

// Mock the fetch function
vi.stubGlobal("fetch", vi.fn());

describe("fetchData tool", () => {
  beforeEach(() => {
    vi.clearAllMocks();
  });

  it("returns parsed data from API", async () => {
    (fetch as any).mockResolvedValueOnce({
      ok: true,
      json: () => Promise.resolve({ items: [{ id: 1 }] }),
    });

    const result = await fetchDataHandler({ query: "test" });
    expect(result.structuredContent.results).toHaveLength(1);
  });

  it("returns error on API failure", async () => {
    (fetch as any).mockResolvedValueOnce({
      ok: false,
      status: 500,
      statusText: "Internal Server Error",
    });

    const result = await fetchDataHandler({ query: "test" });
    expect(result.isError).toBe(true);
  });
});
```

## Python Unit Tests (pytest)

### Testing pure tool functions

```python
# tests/test_tools.py
from your_server.tools import hello, echo, calculate_score


def test_hello_returns_greeting():
    result = hello("World")
    assert result["message"] == "Hello, World! Welcome to MCP."


def test_echo_returns_text_and_timestamp():
    result = echo("test")
    assert result["echo"] == "test"
    assert "timestamp" in result


def test_calculate_score_range():
    result = calculate_score(value=50, max_value=100)
    assert 0 <= result["score"] <= 100


def test_calculate_score_zero():
    result = calculate_score(value=0, max_value=100)
    assert result["score"] == 0
```

### Testing async tools

```python
# tests/test_async_tools.py
import pytest
from unittest.mock import AsyncMock, patch

from your_server.tools import fetch_data


@pytest.mark.asyncio
async def test_fetch_data_success():
    mock_response = AsyncMock()
    mock_response.json.return_value = {"items": [{"id": 1}]}
    mock_response.raise_for_status = lambda: None

    with patch("your_server.tools.httpx.AsyncClient") as mock_client:
        mock_client.return_value.__aenter__.return_value.get.return_value = mock_response
        result = await fetch_data("test query")

    assert len(result["results"]) == 1


@pytest.mark.asyncio
async def test_fetch_data_api_error():
    with patch("your_server.tools.httpx.AsyncClient") as mock_client:
        mock_client.return_value.__aenter__.return_value.get.side_effect = Exception("Connection error")
        result = await fetch_data("test query")

    assert "error" in result
```

### pytest config (pyproject.toml)

Already configured in the template:

```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]

[dependency-groups]
dev = ["pytest>=8.0", "pytest-asyncio>=0.24", "ruff>=0.8"]
```
