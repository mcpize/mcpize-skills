# Social Media Post Templates

Use these templates when generating go-to-market content in Step 4. Replace `{placeholders}` with actual values.

## Twitter/X Thread

```
🚀 Just launched {server_name} — an MCP server that {one_liner_description}.

Connect it to Claude, Cursor, or any MCP client in seconds.

🧵 Thread ↓

1/ The problem: {problem_statement}

{2-3 sentences about pain point}

2/ The solution: {server_name} gives AI agents {count} tools:

{bullet list of tools with one-line descriptions}

3/ Get started in 30 seconds:

npx -y mcpize connect @{username}/{slug} --client claude

Or visit: https://mcpize.com/mcp/{slug}

4/ Built with @MCPize — from idea to published MCP server.

{optional: mention the tech stack, data source, or unique angle}

Try it out and let me know what you think! 🙏
```

## Reddit (r/mcp or domain-specific subreddit)

```
Title: I built {server_name} — {one_liner_description}

Hey r/mcp!

I just published {server_name}, an MCP server that {expanded_description}.

**What it does:**
{bullet list of tools}

**Why I built it:**
{1-2 sentences about the motivation}

**How to try it:**
```
npx -y mcpize connect @{username}/{slug} --client claude
```

Or check it out on MCPize: https://mcpize.com/mcp/{slug}

It's {free/pricing_info}. Would love feedback!
```

## LinkedIn

```
Excited to share {server_name} — an MCP server I just published on MCPize.

{1-2 sentences about what it does and why it matters}

What it provides:
{bullet list of tools}

The Model Context Protocol (MCP) is becoming the standard for connecting AI agents to external tools and data. This server makes it easy to {key_benefit}.

Try it: https://mcpize.com/mcp/{slug}

#MCP #AI #AIAgents #{domain_hashtag}
```

## Hacker News

```
Title: Show HN: {server_name} – {one_liner_description}

Body:
{server_name} is an MCP server that {expanded_description}.

Tools: {comma-separated tool names}

It uses {data_source/API} to provide {key_capability}. {pricing_info}.

Built with MCPize (https://mcpize.com) for deployment and marketplace listing.

https://mcpize.com/mcp/{slug}
```

## Discord

```
🚀 **{server_name}** — {one_liner_description}

Tools: {tool_count} ({comma-separated names})
Pricing: {free/paid}

Quick install:
```
npx -y mcpize connect @{username}/{slug} --client claude
```

https://mcpize.com/mcp/{slug}
```

## README Badge + Install Snippets

### Badge (Markdown)
```markdown
[![Available on MCPize](https://img.shields.io/badge/MCPize-Available-blue)](https://mcpize.com/mcp/{slug})
```

### Connect section (Markdown)
```markdown
## Connect via MCPize

```bash
npx -y mcpize connect @{username}/{slug} --client claude
```

Or visit: https://mcpize.com/mcp/{slug}
```

### Per-client install commands
```
Claude:    claude mcp add --transport http {server_name} {endpoint_url}
Cursor:    cursor mcp add {server_name} {endpoint_url}
Windsurf:  windsurf mcp add {server_name} {endpoint_url}
Cline:     cline mcp add {server_name} {endpoint_url}
```

### JSON Config
```json
{
  "mcpServers": {
    "{server_name}": {
      "url": "{endpoint_url}"
    }
  }
}
```
