---
name: competitor-pricing-tracker
description: |
  Track competitor SaaS pricing, packaging, plan limits, add-ons, discounts,
  and pricing-page changes across one or more companies. Returns a pricing
  matrix, change log, and positioning notes grounded in current website data.
when_to_use: |
  "track competitor pricing", "compare SaaS pricing", "monitor pricing pages",
  "what changed on {company} pricing", "competitor packaging analysis",
  "pricing intelligence for", "pricing page changes".
license: MIT
compatibility: Requires the `composio` CLI on PATH. Connect with `composio link browser_tool` (interactive cloud browser) and `composio link composio_search` (web search and URL fetch).
allowed-tools: Bash Agent
---

# Competitor Pricing Tracker

Track competitor pricing and packaging across SaaS websites. Extracts plans,
prices, feature limits, add-ons, free-trial details, enterprise CTAs, and
pricing-page change signals.

**Output directory**: `~/Desktop/{slug}_competitor_pricing_{YYYY-MM-DD}/`

---

## What It Tracks

### Pricing Page Snapshot
- Plan names and public prices
- Monthly, annual, and usage-based billing terms
- Seat minimums, included usage, and overage pricing
- Feature gates and packaging differences by plan
- Free plan, trial length, credit card requirement, and signup CTA
- Add-ons, platform fees, implementation fees, and support tiers
- Enterprise CTA language, procurement/security claims, and contact path
- Currency, region, tax/VAT notes, and billing disclaimers

### Competitive Positioning
- Target customer implied by each plan
- Core features used as upgrade triggers
- Discount messaging and annual savings claims
- Pricing page copy changes that signal strategic positioning
- Bundled products, usage caps, and expansion levers

### Change Detection
- New, removed, or renamed plans
- Price changes by plan and billing cadence
- Feature moved between tiers
- Trial or free-plan changes
- New add-on, limit, compliance, or enterprise messaging

---

## Data Sources

| Source | What to find | How |
|--------|-------------|-----|
| `{competitor}.com/pricing` | Plans, prices, packaging | `COMPOSIO_SEARCH_FETCH_URL_CONTENT`; fall back to browser |
| `{competitor}.com/plans` or `/compare` | Plan limits and feature matrices | `BROWSER_TOOL_CREATE_TASK` for toggles and accordions |
| Help docs / billing docs | Usage limits, overages, add-ons | `COMPOSIO_SEARCH_WEB`, then fetch matching URLs |
| Signup or checkout flow | Trial details, seat minimums, hidden fees | `BROWSER_TOOL_CREATE_TASK`; stop before payment |
| Blog / changelog | Pricing migration announcements | `COMPOSIO_SEARCH_WEB` and `COMPOSIO_SEARCH_FETCH_URL_CONTENT` |
| G2 / Capterra / review sites | Pricing complaints and buyer language | `COMPOSIO_SEARCH_WEB`; use browser only for public pages |
| Archived snapshots | Historical pricing if user provides URLs | Fetch archived page; browser fallback for rendered captures |

Prefer direct company sources over third-party summaries. Use third-party
pages only to fill gaps or identify claims to verify against the vendor site.
