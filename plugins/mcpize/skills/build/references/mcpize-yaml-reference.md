# mcpize.yaml Reference

`mcpize.yaml` is the deployment config for your MCP server — it tells MCPize how to build and run it.

## Minimal Config

### TypeScript
```yaml
version: 1
runtime: typescript
entry: src/index.ts
build:
  install: npm ci
  command: npm run build
startCommand:
  type: http
  command: node dist/index.js
configSchema:
  source: code
```

### Python
```yaml
version: 1
runtime: python
entry: src/your_server/server.py
build:
  install: uv sync --frozen --no-dev
startCommand:
  type: http
  command: .venv/bin/python -m your_server.server
configSchema:
  source: code
```

> **Note**: `mcpize init` automatically sets the package name in `entry` and `startCommand.command` via the post-init script. You don't need to update these paths manually.

## All Fields

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `version` | Yes | `1` | Always `1` |
| `name` | No | string | Server name (auto-detected from package.json if omitted) |
| `description` | No | string | Short description of what the server does |
| `runtime` | Yes | enum | `typescript`, `python`, `php`, or `container` |
| `entry` | Yes | string | Main source file path (e.g., `src/index.ts`) |

### Build

| Field | Required | Description |
|-------|----------|-------------|
| `build.install` | Yes | Dependency install command (e.g., `npm ci`, `uv sync --frozen --no-dev`) |
| `build.command` | No | Build/compile command (e.g., `npm run build`). Not needed if no build step. |
| `build.dockerfile` | No | Path to Dockerfile (only for `runtime: container`) |

### Start Command

| Field | Required | Description |
|-------|----------|-------------|
| `startCommand.type` | Yes | **Always use `http`** for MCPize. Options: `http`, `sse`, `stdio` |
| `startCommand.command` | Yes | How to start the server (e.g., `node dist/index.js`) |
| `startCommand.args` | No | Extra CLI arguments as array |

**Important**: Always use `type: http` (Streamable HTTP). This is the modern MCP transport and the only one recommended for MCPize Cloud deployment.

### Config Schema

```yaml
configSchema:
  source: code   # MCPize auto-discovers tools from your server
```

Always include this — it tells MCPize to auto-detect your tools, resources, and prompts after deployment.

## Secrets (Publisher-Owned)

Secrets are API keys that YOU (the server publisher) provide. They're encrypted and stored securely.

```yaml
secrets:
  - name: OPENAI_API_KEY
    required: true
    description: OpenAI API key for completions
  - name: DATABASE_URL
    required: true
    description: Postgres connection string
  - name: OPTIONAL_KEY
    required: false
    description: Optional analytics API key
```

Set them via CLI: `mcpize secrets set OPENAI_API_KEY sk-xxx`

In your code, read with `process.env.OPENAI_API_KEY` (TS) or `os.getenv("OPENAI_API_KEY")` (Python).

## Credentials (Subscriber BYOK)

Credentials are API keys that each subscriber brings themselves. Use this when your server acts as a proxy to another service and users need their own access.

```yaml
credentials:
  - name: GITHUB_TOKEN
    required: true
    description: Your GitHub personal access token
    docs_url: https://github.com/settings/tokens
    mapping:
      env: GITHUB_TOKEN        # Injected as environment variable
      header: X-GitHub-Token   # Also available as HTTP header
credentials_mode: per_user      # Each subscriber has their own credentials
```

### Credential mapping options

| Mapping | How it works |
|---------|-------------|
| `env: VAR_NAME` | Set as environment variable in your server |
| `header: X-Header-Name` | Passed as HTTP header to your server |
| `arg: --flag-name` | Added as CLI argument (for stdio servers) |

### When to use secrets vs credentials

| | Secrets | Credentials |
|-|---------|-------------|
| **Who provides** | You (the publisher) | Each subscriber |
| **When set** | At deploy time | When user subscribes |
| **Example** | Your OpenAI key for server logic | User's GitHub token |
| **Where in yaml** | `secrets:` | `credentials:` + `credentials_mode: per_user` |

## Example: Full Config with Secrets + Credentials

```yaml
version: 1
runtime: typescript
entry: src/index.ts

build:
  install: npm ci
  command: npm run build

startCommand:
  type: http
  command: node dist/index.js

configSchema:
  source: code

# Your infrastructure secrets
secrets:
  - name: OPENAI_API_KEY
    required: true
    description: OpenAI API key for AI analysis features

# Per-user credentials (BYOK)
credentials:
  - name: GITHUB_TOKEN
    required: true
    description: GitHub personal access token with repo scope
    docs_url: https://github.com/settings/tokens
    mapping:
      env: GITHUB_TOKEN
credentials_mode: per_user
```
