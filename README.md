# MCPize Skills

**Go from "what should I build?" to a live MCP server on the marketplace — in one session.**

[![MCPize](https://img.shields.io/badge/MCPize-Suite-blue)](https://mcpize.com)

Three skills. One pipeline. You choose the idea, we handle the rest.

## What's in the box

| Skill | The vibe |
|-------|----------|
| `/mcpize:idea` | Your brainstorming buddy. Interviews you about your skills & domain, generates 3-5 tailored MCP server ideas, researches competitors, validates APIs, runs monetization analysis, and hands you a ready-to-build brief. |
| `/mcpize:build` | Your pair programmer. Takes the brief, scaffolds a project via `mcpize init`, implements every tool with proper validation and error handling, writes tests, runs a live verification loop, and makes sure everything actually works. |
| `/mcpize:publish` | Your launch partner. Deploys to MCPize Cloud, generates SEO metadata + logo + pricing via AI, publishes to the marketplace, creates social media posts, and verifies the live endpoint. Ship day feels good. |

## Get started

### Claude Code

```bash
/plugin marketplace add mcpize/mcpize-skills
/plugin install mcpize@mcpize-skills
```

That's it. Now run:

```
/mcpize:idea      # "I want to build something but don't know what"
/mcpize:build     # "Here's my brief, let's code"
/mcpize:publish   # "Ship it!"
```

**One-shot** — let the skills chain automatically:

```
> Using /mcpize:idea, /mcpize:build, and /mcpize:publish —
  I'm a TypeScript dev, build me an MCP server for SEO audits and ship it.
```

Idea discovery, build, deploy, publish — all in one prompt.

### Cursor, Copilot, Windsurf, and 40+ other agents

```bash
skillkit install mcpize/mcpize-skills
```

[SkillKit](https://www.agenstskills.com/) auto-translates the skills into whatever format your agent needs.

### Manual install

Grab the `skills/` folder and drop it into your agent's skill directory:
- **Claude Code**: `~/.claude/skills/`
- **Cursor**: `.cursor/skills/`
- **Copilot**: `.github/skills/`

## The pipeline

```
  /mcpize:idea              /mcpize:build              /mcpize:publish

  "What should I build?"    "Let's code this thing"    "Time to ship"
         |                         |                          |
         v                         v                          v
  Interview you             Parse brief & plan         Quality checks
  Generate ideas            Scaffold project           Deploy to cloud
  Research competitors      Implement all tools        AI-generate SEO + logo
  Validate APIs             Add caching & errors       Set up pricing
  Analyze monetization      Write tests                Publish listing
         |                  Verify everything                 |
         v                         |                          v
  mcp-brief-*.md                   v                   Live on mcpize.com
  (ready for build)         Working server             + social posts
                            (ready for publish)        + launch checklist
```

Each skill picks up where the last one left off. Run all three in sequence, or jump in wherever you are.

## What's inside

```
mcpize-skills/
  .claude-plugin/
    plugin.json            # Plugin manifest
    marketplace.json       # Marketplace catalog
  skills/
    idea/
      SKILL.md             # The brainstorming skill
      references/
        mcp-landscape.md   # Market data, niches, monetization models
    build/
      SKILL.md             # The builder skill
      references/
        typescript-patterns.md
        python-patterns.md
        mcpize-yaml-reference.md
        testing-patterns.md
    publish/
      SKILL.md             # The publisher skill
      references/
        quality-checklist.md
        social-templates.md
  .skills                  # SkillKit manifest (cross-IDE)
  package.json             # npm distribution
```

## You'll need

- **[mcpize CLI](https://mcpize.com)** — `npm install -g mcpize`
- **Node.js 18+** or **Python 3.11+** (depending on your server's language)

## Built with MCPize

Part of the [MCPize](https://mcpize.com) ecosystem. MCPize gives developers 85% revenue share and handles hosting, payments, and discovery. These skills are the fastest path from zero to a published, monetizable MCP server.

## Examples

### Step by step — run each skill when you're ready

For the pros who like full control:

```
> /mcpize:idea
  I'm a Python dev who trades crypto. I want passive income.

  ... interviews you, generates 5 ideas, you pick "DeFi Yield Analyzer",
  competitive research, API validation, monetization analysis ...
  -> saves mcp-brief-defi-yield-analyzer.md

> /mcpize:build
  ... reads the brief, scaffolds project, implements 4 tools,
  adds caching + error handling, tests pass, live verification green ...

> /mcpize:publish
  ... deploys to MCPize Cloud, AI generates SEO + logo + pricing,
  publishes listing, creates Twitter thread + Reddit post ...
  -> live at mcpize.com/mcp/defi-yield-analyzer
```

## License

MIT — do whatever you want with it.
