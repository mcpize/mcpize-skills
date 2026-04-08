# Social Media Post Templates

Use these templates when generating go-to-market content in Step 4. Replace `{placeholders}` with actual values.

**IMPORTANT — viral-pattern rules (backed by 2025 X algorithm research):**
- Retweets count 20x, replies 13.5x, bookmarks 10x, likes 1x. **External links cost -30 to -50% reach.**
- Therefore: **link goes in REPLY, not in main tweet**
- Casual cali-dev tone: lowercase, self-deprecating, specific numbers, short sentences
- 1-2 functional emojis MAX (no rocket/fire walls, no 🚀, no 🧵 thread bait)
- Hook formulas: vulnerability/confession, contrarian declaration, bold specific numbers, pattern interrupt
- NO corporate openers ("Excited to share!", "Just launched!", "🚀 Introducing...")

---

## Twitter/X — Pick One of 3 Variants

### Variant A — "Builder's confession" (vulnerability hook)

**Use when:** server is in a niche/contrarian space, or user wants to challenge conventional wisdom

```
everyone keeps overcomplicating {domain}

i had no plan. no spec. no clue what to build

{X} minutes later i shipped {server_name} — {one_liner}

every keystroke. unedited
```

**Reply (post separately after main tweet):**
```
built with @MCPize → mcpize.com/mcp/{slug}

what would you build with it?
```

---

### Variant B — "Eat your own cooking" (creator angle)

**Use when:** user is also the tool/skill author (strongest trust signal)

```
built {server_name} to {key_benefit}

then used it on myself for {use_case}

{specific_result_with_number}

it's open source btw
```

**Reply:**
```
grab it → mcpize.com/mcp/{slug}

taking requests for the next build
```

---

### Variant C — "Specific number flex"

**Use when:** there are concrete metrics to flex (tool count, time, data points, etc.)

```
{number} {tools/things} in {timeframe}

{server_name}: {one_liner}

no framework. no boilerplate. just {data_source} + Claude

still processing that this works tbh
```

**Reply:**
```
→ mcpize.com/mcp/{slug}

drop your idea below, i might build it next
```

---

## Click-to-Tweet URL (Twitter Intent API)

After picking a variant and filling placeholders, generate a clickable URL the user can click ONCE to pre-fill their tweet.

**Format:**
```
https://twitter.com/intent/tweet?text=URL_ENCODED_TWEET_TEXT
```

**How to construct (Claude does this automatically):**
1. Take the chosen variant's main tweet body (NOT the reply — link is separate)
2. URL-encode the text using JavaScript-style encoding:
   - Newlines → `%0A%0A` (double newline for paragraph break)
   - Spaces → `%20`
   - `@` → `%40`
   - `:` → `%3A`
   - `/` → `%2F`
   - `—` → `%E2%80%94`
3. Present to user as a markdown clickable link:
   ```
   [📤 Click to publish on X](https://twitter.com/intent/tweet?text=ENCODED_HERE)
   ```

**Critical:**
- Do NOT include `mcpize.com/mcp/{slug}` in the encoded text (link in main tweet = -30-50% reach penalty)
- The user posts the link in a REPLY after the main tweet goes live
- Always show the reply text separately so user can copy-paste it after publishing

**Example (filled in):**

Main tweet (variant B for a tool called "FDA Drug Info"):
```
built fda-drug-info to query 50k+ drug labels in one call

then used it to debug my mom's prescription interactions

found 2 issues her pharmacist missed

it's open source btw
```

Encoded URL:
```
https://twitter.com/intent/tweet?text=built%20fda-drug-info%20to%20query%2050k%2B%20drug%20labels%20in%20one%20call%0A%0Athen%20used%20it%20to%20debug%20my%20mom%27s%20prescription%20interactions%0A%0Afound%202%20issues%20her%20pharmacist%20missed%0A%0Ait%27s%20open%20source%20btw
```

Reply text (copy-paste after posting):
```
grab it → mcpize.com/mcp/fda-drug-info

taking requests for the next build
```

---

## Reddit (r/mcp or domain-specific subreddit)

Reddit doesn't penalize links — keep the link in body. Drop the corporate "Hey r/mcp!" opener, lead with the build story.

```
Title: built {server_name} — {one_liner_description}

just shipped {server_name}, an MCP server that {expanded_description}.

**what it does:**
{bullet list of tools}

**why i built it:**
{1-2 sentences — start with a real problem you hit, not "I wanted to learn MCP"}

**how to try it:**
\```
npx -y mcpize connect @{username}/{slug} --client claude
\```

or check it on MCPize: https://mcpize.com/mcp/{slug}

it's {free/pricing_info}. would love feedback — what should i add next?
```

---

## LinkedIn

Drop "Excited to share!" — use a builder voice. Specific numbers, real story.

```
shipped {server_name} this week — an MCP server that {does_what}.

the backstory: {1-2 sentences about the actual problem you hit}

what's inside:
{bullet list of tools, max 4}

the Model Context Protocol is becoming the standard for connecting AI agents to external tools. this server makes it dead simple to {key_benefit}.

try it: https://mcpize.com/mcp/{slug}

#MCP #AI #AIAgents #{domain_hashtag}
```

---

## Hacker News

```
Title: Show HN: {server_name} – {one_liner_description}

Body:
{server_name} is an MCP server that {expanded_description}.

I built it because {real reason — be specific, HN sees through marketing fluff}.

Tools: {comma-separated tool names}

It uses {data_source/API} to provide {key_capability}. {pricing_info}.

The Model Context Protocol lets AI agents (Claude, Cursor, etc.) call this as a tool with one config line.

Built with MCPize (https://mcpize.com) for hosting and listing.

https://mcpize.com/mcp/{slug}

Happy to answer questions about {tech_stack_or_domain}.
```

---

## Discord

```
just shipped **{server_name}** — {one_liner_description}

tools: {tool_count} ({comma-separated names})
pricing: {free/paid}

quick install:
\```
npx -y mcpize connect @{username}/{slug} --client claude
\```

https://mcpize.com/mcp/{slug}

would love any feedback 🙏
```

---

## README Badge + Install Snippets

### Badge (Markdown)
```markdown
[![Available on MCPize](https://img.shields.io/badge/MCPize-Available-blue)](https://mcpize.com/mcp/{slug})
```

### Connect section (Markdown)
```markdown
## Connect via MCPize

\```bash
npx -y mcpize connect @{username}/{slug} --client claude
\```

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
