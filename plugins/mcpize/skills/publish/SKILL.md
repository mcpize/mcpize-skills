---
name: publish
description: "MCPize Publisher — takes a built MCP server and publishes it to the MCPize marketplace with full autopilot. Runs quality checks, deploys to MCPize Cloud, generates SEO metadata and logo via AI, sets up pricing, publishes to marketplace, generates go-to-market content (social posts, README badges, launch checklist), and verifies the live listing. Part of the MCPize suite (mcpize.com). Use this skill whenever someone wants to publish, launch, or list an MCP server on the marketplace, wants to deploy and publish, says 'publish this MCP', 'launch my server', 'put it on the marketplace', or references mcpize-publish. Also trigger on: publish mcp, deploy mcp, launch mcp server, list on marketplace, mcpize publish, go live, ship it, publish to mcpize, marketplace listing, go-to-market mcp."
---

# MCPize: Publish Your MCP Server

> Part of the **MCPize** suite (https://mcpize.com) — from idea to published MCP server.
>
> **Previous**: `/mcpize:idea` (discover & validate) → `/mcpize:build` (build & test)
> **This skill**: Publish to marketplace + Go-to-Market

Take a built MCP server and publish it to the MCPize marketplace — SEO, pricing, logo, listing, social media posts, all in one flow.

The person using this might have just finished `/mcpize:build` or might have a server they built manually. Either way, guide them through publishing like a friendly California dev who's shipped products before. Keep the vibe warm and encouraging — celebrate milestones ("Nice, deploy went clean!"), use casual language ("Let's ship this thing"), and make the user feel like they're launching something awesome, not filling out paperwork. Professional but human.

### Reference files

Before starting, read these reference files for details:
- `${CLAUDE_SKILL_DIR}/references/quality-checklist.md` — 16 quality checks the dashboard uses (C1-C5, I1-I6, N1-N5) and what `--auto` handles
- `${CLAUDE_SKILL_DIR}/references/social-templates.md` — ready-to-fill templates for Twitter, Reddit, LinkedIn, HN, Discord, README badges

---

## The Job

0. **Locate project & parse context** — find the server and its brief
1. **Quality gate** — make sure it's ready (skip if `/mcpize:build` already verified)
2. **Deploy** — get it on MCPize Cloud (skip if already deployed)
3. **Publish** — SEO + pricing + logo + marketplace listing via `mcpize publish --auto`
4. **Go-to-Market + verify** — social posts, README updates, launch checklist, live verification

---

## Step 0: Locate Project & Parse Context

### Find the server

Look for `mcpize.yaml` in the current directory. If not found, check subdirectories one level deep.

If multiple candidates or none found, use `AskUserQuestion`:
- **header**: "Which project should I publish?"
- **subheader**: "I need to find your MCP server project"
- Provide options or ask for a path

### Find the brief

Look for `mcp-brief-*.md` in the project directory or parent directory. The brief contains:
- **Pricing model** → determines `--free` vs `--pricing "..."`
- **Server description** → enriches SEO generation
- **Tool list** → for social media content
- **Target audience** → for marketing copy

If no brief exists, extract context from:
1. `README.md` — description, tools, features
2. `package.json` / `pyproject.toml` — name, description
3. `mcpize.yaml` — runtime, secrets
4. Source code — tool names and descriptions from MCP registration

### Extract requirements

Build a mental model:
- **Server name** (from mcpize.yaml, package.json, or directory name)
- **Description** (from brief or README)
- **Tools list** (from brief or source code)
- **Pricing model** (from brief: "Free", "Freemium", "Paid")
- **Runtime** (TypeScript or Python — from mcpize.yaml)
- **Required secrets** (from mcpize.yaml `secrets:` section)

---

## Step 1: Pre-Publish Quality Gate

### Skip logic (IMPORTANT)

**If `/mcpize:build` ran earlier in this conversation** and all checks passed (tests, smoke test, doctor all green), **skip most checks but still verify login**:

1. Run `mcpize whoami` — if fails, tell user to run `mcpize login` and wait
2. Then note: "Quality checks already passed during `/mcpize:build` — skipping to deploy."

Only run checks if:
- No build history in this conversation
- User built the server manually
- User explicitly asks to re-verify

### Checks to run (when needed)

Run these in order. Stop on first blocker failure.

| # | Check | Command | Blocker? |
|---|-------|---------|----------|
| 1 | Logged in | `mcpize whoami` | YES — always check, even after build |
| 2 | Tests pass | `npm test` or `pytest` | YES |
| 3 | Smoke test | `npm run test:smoke` or `bash test-mcp.sh` | YES |
| 4 | Doctor check | `mcpize doctor` | YES |
| 5 | README exists | Check file exists and >200 chars | YES |
| 6 | mcpize.yaml valid | Parse YAML, check required fields | YES |
| 7 | No hardcoded secrets | Use Grep tool to search `src/` for patterns like `sk-`, `api_key.*=` | Warning only |

If a blocker fails:
1. Show the error clearly
2. Suggest the fix
3. Help implement the fix
4. Re-run the check
5. Continue only when it passes

**Login failure**: If `mcpize whoami` fails, tell the user to run `mcpize login` in their terminal (it opens a browser for auth). Stop and wait — do NOT proceed without auth. After they confirm login, re-run `mcpize whoami` to verify.

---

## Step 2: Deploy (if needed)

### Check current deployment status

First, check if `.mcpize/project.json` exists in the project directory. If it does NOT exist, the server has never been deployed — skip `mcpize status` and proceed directly to deploy.

If `.mcpize/project.json` exists, check health:
```bash
mcpize status
```

If the server is already deployed and healthy → **skip this step**:
> "Server already deployed and healthy — skipping deploy."

### Deploy to MCPize Cloud

This is an **irreversible action**. Follow the protocol:

1. **State WHY**: "Deploying creates a live endpoint on MCPize Cloud that users can connect to. This makes your server accessible via `https://{name}.mcpize.run`."

2. **Tell user what will happen**:
   - Code will be uploaded and built in the cloud
   - A live endpoint will be created
   - Health monitoring will start

3. **Warn the user**: "Deploy takes 3-5 minutes — your code gets uploaded, built in the cloud, and a live endpoint is created. Grab a coffee."

   Then share a wisdom quote about patience. Pick one that fits the moment — something original, not overused. Examples:
   - "The two most powerful warriors are patience and time." — Leo Tolstoy
   - "Nature does not hurry, yet everything is accomplished." — Lao Tzu
   - "Have patience. All things are difficult before they become easy." — Saadi

4. **Run deploy** (with `--skip-wizard` to control publish separately):
```bash
mcpize deploy --skip-wizard --yes
```

> **Note**: Deploy creates the server with default "free" status — this is expected. The server is NOT visible on the marketplace yet (deploy = infrastructure only, publish = marketplace listing). Pricing gets set correctly in Step 3 during `mcpize publish`. Tell the user: "Don't worry about the 'free' status — that's just a deploy default. We'll set up real pricing in the next step."

4. **Verify deployment**:
```bash
mcpize status
```

Confirm the status shows "ready" or "healthy".

5. **If deploy fails**: Run `mcpize diagnose` and help debug.

---

## Step 3: Publish to Marketplace

### Determine pricing from brief

Read the brief's Monetization section:
- If "Free" or no pricing mentioned → use `--auto` (defaults to free)
- If "Freemium" or "Paid" → use `--auto --pricing "description from brief"`

**CRITICAL — MCPize billing constraints (do NOT invent features that don't exist):**

MCPize supports exactly 3 plan types:

| Type | How it works | Example |
|------|-------------|---------|
| `fixed` | Flat monthly/yearly subscription | Pro $19/mo, 500 requests/day |
| `usage` | Pure pay-per-use, no base fee | $0.01 per request |
| `hybrid` | Base subscription + overage pricing | $19/mo for 500 req + $0.005/req over limit |

**Quotas are per-request and per-token only.** Rate limits are per second/minute/hour/day.

**DO NOT generate pricing with:**
- ❌ Seats / per-user pricing (MCPize bills per API request, not per user)
- ❌ "Team" or "Enterprise" tiers with seats, SSO, HIPAA, BAA, SLA
- ❌ "Contact sales" / "Custom pricing" options
- ❌ Feature gating by tool name (all tools are available on all plans)
- ❌ Any capability MCPize doesn't actually support

**Valid pricing patterns:**
```
# Freemium
"Free: 50 requests/day. Pro $9.99/mo: 1000 requests/day"

# Usage-based
"Pay per use: $0.01 per request, first 100 free"

# Hybrid
"Starter $9.99/mo: 500 requests/day. Pro $29/mo: 2000 requests/day, $0.005/req overage"
```

**Rules:**
- Maximum 1 free plan (price_monthly: 0)
- Paid plans must have price_monthly > 0
- Unlimited quota only if price >= $50/mo OR usage/hybrid type with usage_price > 0
- Differentiate plans by request quotas, rate limits, and token limits — not by features or seats

### Preview first (optional but recommended)

```bash
mcpize publish --dry-run --auto
# or
mcpize publish --dry-run --auto --pricing "Free: 50 requests/day. Pro $9.99/mo: 1000 requests/day"
```

Show the user what will happen before executing.

### Run publish

For **free servers**:
```bash
mcpize publish --auto
```

For **paid servers**:
```bash
mcpize publish --auto --pricing "{pricing description from brief}"
```

This single command does:
1. Clean server name via AI (e.g., "fda-drug-info-mcp" → "FDA Drug Info")
2. Generate SEO content (display name, category, tags, descriptions) via AI
3. Set up pricing (free or AI-generated tiers)
4. Generate logo via AI (OpenAI gpt-image-1)
5. Publish to MCPize marketplace

**If publish fails with "Failed to update server status"** but name/SEO/pricing/logo all succeeded — check `mcpize status`. If the server is already `active`, this is expected (deploy already made it live). Just verify with `mcpize publish --show` that SEO and pricing are configured, and move on.

### Verify publish

```bash
mcpize publish --show
```

Confirm SEO content is configured and pricing is set.

---

## Step 4: Go-to-Market + Verification

### 4a: Generate viral-pattern social posts

Use templates from `${CLAUDE_SKILL_DIR}/references/social-templates.md`. The templates apply 2025 X algorithm research — DO NOT fall back to old "🚀 Just launched / 🧵 Thread ↓ / features list" format.

**Twitter/X — viral pattern (CRITICAL)**

1. **Pick the right variant** (A/B/C) from social-templates.md based on the server's story:
   - **Variant A** (Builder's confession) — niche/contrarian servers, vulnerability angle
   - **Variant B** (Eat your own cooking) — when user is also the tool/skill author (strongest trust signal)
   - **Variant C** (Specific number flex) — when there are concrete metrics (tool count, time, data points)

2. **Fill placeholders** with actual server data: `{server_name}`, `{slug}`, `{one_liner}`, `{tool_count}`, `{key_benefit}`, etc.

3. **Generate a Twitter Intent URL** so the user clicks once to publish:
   - Take the chosen variant's main tweet body (NOT the reply — the marketplace link goes in a separate reply to avoid the -30-50% reach penalty for external links)
   - URL-encode: spaces → `%20`, newlines → `%0A%0A`, `@` → `%40`, `:` → `%3A`, `/` → `%2F`, `'` → `%27`, `+` → `%2B`, `—` → `%E2%80%94`
   - Format: `https://twitter.com/intent/tweet?text={encoded_text}`

4. **Present to the user as a clickable markdown link** plus the reply text separately so they can copy-paste it after the main tweet goes live.

**Output format (show this to the user):**

```
🐦 Your viral tweet is ready (variant B — builder/creator angle)

Main tweet:
─────────────────
built fda-drug-info to query 50k+ drug labels in one call

then used it to debug my mom's prescription interactions

found 2 issues her pharmacist missed

it's open source btw
─────────────────

[📤 Click to publish on X](https://twitter.com/intent/tweet?text=built%20fda-drug-info%20to%20query%2050k%2B...)

After posting, reply to your own tweet with:
─────────────────
grab it → mcpize.com/mcp/fda-drug-info

taking requests for the next build
─────────────────

Why this works:
• Vulnerability + specific numbers (50k+, 2 issues) → high dwell time
• Link in REPLY not main tweet → avoids -30-50% reach penalty
• Question CTA in reply → triggers replies (13.5x algo weight)
```

**Other platforms** (Reddit, LinkedIn, HN, Discord) — generate using their respective templates from social-templates.md. These don't penalize links the way X does, so links stay in body. Tone still casual builder voice — no "Excited to share!" corporate openers.

### 4b: Update README with MCPize badges and install snippets

Use the templates from `${CLAUDE_SKILL_DIR}/references/social-templates.md` (section "README Badge + Install Snippets") to add:

1. **MCPize badge** — shields.io badge linking to marketplace page
2. **Connect section** — `npx -y mcpize connect @{username}/{slug} --client claude`
3. **Per-client install commands** — Claude, Cursor, Windsurf, Cline CLI commands
4. **JSON config** — for manual setup

Get the endpoint URL from `mcpize status` output. Get the slug from the server info.

### 4c: Generate launch checklist

Create `LAUNCH-CHECKLIST.md` in the project directory:

```markdown
# Launch Checklist: {Server Name}

## Pre-Launch
- [x] Server deployed and healthy
- [x] Marketplace listing published
- [x] SEO metadata configured
- [x] Pricing set
- [x] Logo generated
- [ ] README updated with MCPize badge + install snippets
- [ ] GitHub repo public (if applicable)

## Launch Day
- [ ] Twitter/X thread posted
- [ ] Reddit r/mcp post published
- [ ] Discord communities notified
- [ ] LinkedIn post shared

## Week 1
- [ ] Hacker News "Show HN" posted
- [ ] Monitor MCPize dashboard for first subscribers
- [ ] Check `mcpize logs` for errors
- [ ] Respond to user feedback

## Week 2-4
- [ ] Dev.to / Hashnode tutorial published
- [ ] Add features based on usage patterns
- [ ] Consider blog post about building it
```

### 4d: Verification loop ("Ralph loop")

Verify the published server works end-to-end:

1. **Deployment health**:
```bash
mcpize status
```
Confirm status is healthy.

2. **MCP protocol test** to live endpoint (hosted servers require auth):
```bash
# Get auth token and endpoint URL
TOKEN=$(python3 -c "import json; print(json.load(open('$HOME/.mcpize/config.json'))['token'])")

# Health check (no auth needed)
curl -s {endpoint}/health

# MCP initialize (auth required)
curl -s {endpoint}/mcp \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-03-26","capabilities":{},"clientInfo":{"name":"verify","version":"1.0.0"}}}'
```

3. **Tools verification**:
```bash
curl -s {endpoint}/mcp \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/list","params":{}}'
```
Confirm all expected tools are listed.

4. **Error log check**:
```bash
mcpize logs --severity ERROR
```
Should return no recent errors.

5. **If any check fails**:
   - Diagnose the issue (`mcpize diagnose` or manual investigation)
   - Fix the problem
   - Re-deploy if code changes were needed: `mcpize deploy --skip-wizard --yes`
   - Re-run verification from step 1
   - Only proceed when ALL checks pass

### 4e: Final handoff

Present the summary:

```
Published! Your MCP server is live.

Server:   {name}
Listing:  https://mcpize.com/mcp/{slug}
Endpoint: {endpoint-url}
Tools:    {count} tools
Pricing:  Free / ${price}/mo

What's done:
  [x] Deployed to MCPize Cloud
  [x] SEO metadata generated and saved
  [x] Logo generated
  [x] Pricing configured
  [x] Published to marketplace
  [x] Verified — all tools responding

Next steps:
  1. Review your listing: https://mcpize.com/mcp/{slug}
  2. Upload a custom logo if you want (dashboard > General tab)
  3. Post the social media content below
  4. Follow the LAUNCH-CHECKLIST.md

Social media posts:
  [Include all generated posts for easy copy-paste]
```

---

## Tone & Communication Style

Talk like a friendly California dev who's genuinely stoked to help ship something. Examples:
- After deploy succeeds: "Nice, deploy went clean! Server's live and healthy."
- Before publish: "Alright, let's get this on the marketplace. Here's what's about to happen..."
- After everything's done: "Boom, you're live! Your server is on the marketplace and ready for users."
- On errors: "Hmm, that didn't go as planned. Let me dig into this..." (stay calm, debug, fix)

Don't overdo it — keep it professional but warm. No forced enthusiasm, just genuine helpfulness.

## Important Notes

- **Never publish without user awareness** — always show `--dry-run` first or clearly state what will happen
- **Deploy and publish are separate actions** — deploy puts code in the cloud, publish makes the listing visible on the marketplace
- **Logo generation uses OpenAI** — it costs money on the platform side. If it fails (rate limit, payment issue), continue without logo and note it can be added later in the dashboard
- **Social media content is generated, not posted** — the user decides when and where to post
- **This should feel like a launch party, not a chore** — celebrate the achievement!
