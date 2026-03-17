# Server Quality Checklist

Mirror of `server-quality.ts` — the same checks the MCPize dashboard uses to evaluate listing completeness.

## Scoring

- **Critical** (15 pts each) — must pass for listing to work
- **Important** (3 pts each) — makes listing look professional
- **Nice-to-have** (1.4 pts each) — polish and trust signals

Levels: `incomplete` → `basic` → `good` (60-85%) → `excellent` (85%+)

## Critical Checks

| ID | Check | What passes | Dashboard tab |
|----|-------|------------|---------------|
| C1 | Pricing configured | At least 1 plan (paid servers only) | Monetize |
| C3 | Server reachable | Endpoint exists AND health != error/down | Health |
| C4 | Tools discovered | At least 1 tool registered | Capabilities |
| C5 | Real name | Not "Unnamed Server", "test-server", etc. | General |

## Important Checks

| ID | Check | What passes | Dashboard tab |
|----|-------|------------|---------------|
| I1 | Logo uploaded | `logo_url` is set | General |
| I2 | Description quality | `description` >= 50 characters | General |
| I3 | Category set | Category != null and != "Other" | General |
| I4 | Tags added | At least 1 tag | General |
| I5 | Long description | `long_description` > 100 characters | General |
| I6 | Tool descriptions | All tools have descriptions | Capabilities |

## Nice-to-Have Checks

| ID | Check | What passes | Dashboard tab |
|----|-------|------------|---------------|
| N1 | Website/GitHub link | `website` or `github_url` set | General |
| N2 | SEO metadata | Both `seo_title` and `seo_description` set | SEO |
| N3 | Documentation | `documentation_enabled` AND `documentation` content | Docs |
| N4 | Health check active | `health_status` != unknown/null | Health |
| N5 | Multiple pricing tiers | 2+ plans (paid servers only) | Monetize |

## What `mcpize publish --auto` handles

| Check | Handled by --auto? |
|-------|-------------------|
| C1 Pricing | ✅ marks free or generates plans |
| C3 Reachable | ❌ requires prior `mcpize deploy` |
| C4 Tools | ❌ auto-discovered after deploy |
| C5 Real name | ✅ AI generates display name |
| I1 Logo | ✅ AI generates logo |
| I2 Description | ✅ AI generates short description |
| I3 Category | ✅ AI picks category |
| I4 Tags | ✅ AI generates tags |
| I5 Long description | ✅ AI generates long description |
| I6 Tool descriptions | ❌ must be in code (done during build) |
| N1 Website/GitHub | ❌ manual — suggest in handoff |
| N2 SEO metadata | ✅ AI generates (via saveSEO) |
| N3 Documentation | ❌ manual — suggest in handoff |
| N4 Health check | ❌ auto after deploy |
| N5 Multiple tiers | ✅ if --pricing used with multiple tiers |
