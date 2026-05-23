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

---

## Pipeline

### Step 1: Discover pricing URLs

Use search when the user gives company names instead of exact URLs:

```bash
for COMPANY in {COMPETITOR_NAMES}; do
  for Q in \
    "$COMPANY pricing" \
    "$COMPANY plans pricing" \
    "$COMPANY billing usage limits"; do
    composio execute COMPOSIO_SEARCH_WEB -d "$(jq -n --arg q "$Q" '{ query: $q }')"
  done
done
```

Keep the canonical pricing page, plan comparison page, billing docs, and
recent pricing-announcement posts for each competitor.

### Step 2: Fetch static pricing pages first

Fetch is faster and works well for static HTML pricing pages:

```bash
composio execute COMPOSIO_SEARCH_FETCH_URL_CONTENT -d "$(jq -n \
  --argjson urls "$(printf '%s\n' "${PRICING_URLS[@]}" | jq -R . | jq -s .)" \
  '{ urls: $urls, max_characters: 30000 }')"
```

If the returned text misses prices, plan cards, feature matrices, billing
toggles, or accordion content, use the cloud browser.

### Step 3: Extract JS-rendered pricing with Composio browser

```bash
composio execute BROWSER_TOOL_CREATE_TASK -d '{
  "task": "Extract the full SaaS pricing page. Toggle monthly and annual billing if present. Expand feature comparison rows, FAQ accordions, add-on sections, and enterprise/contact-sales sections. For each plan capture: name, monthly price, annual price, currency, billing unit, seat minimum, included usage, overages, feature limits, trial/free-plan terms, add-ons, CTA text, and any disclaimers. Return normalized JSON plus short notes on ambiguous values.",
  "startUrl": "{PRICING_URL}"
}'
```

For checkout or signup flows, stop before submitting account, payment, or
contract information:

```bash
composio execute BROWSER_TOOL_CREATE_TASK -d '{
  "task": "Open the signup or checkout path only far enough to inspect public pricing, trial terms, required seat count, and payment requirements. Do not create an account, submit personal data, enter payment details, or accept terms. Return the visible pricing details as JSON.",
  "startUrl": "{SIGNUP_OR_CHECKOUT_URL}"
}'
```

### Step 4: Compare competitors in a batch

```bash
mkdir -p ~/Desktop/{slug}_competitor_pricing_{YYYY-MM-DD}/raw

for URL in "${PRICING_URLS[@]}"; do
  SAFE_NAME=$(printf "%s" "$URL" | tr -cs '[:alnum:]' '_' | sed 's/_$//')
  composio execute BROWSER_TOOL_CREATE_TASK -d "$(jq -n --arg u "$URL" '{
    task: "Extract SaaS pricing, packaging, plan limits, add-ons, trial details, enterprise CTA, and disclaimers. Toggle billing options and expand hidden sections. Return normalized JSON.",
    startUrl: $u
  }')" > ~/Desktop/{slug}_competitor_pricing_{YYYY-MM-DD}/raw/"$SAFE_NAME.json"
done
```

### Step 5: Check recent pricing changes

```bash
for COMPANY in {COMPETITOR_NAMES}; do
  for Q in \
    "$COMPANY pricing change 2025 2026" \
    "$COMPANY new pricing plans" \
    "$COMPANY billing changelog"; do
    composio execute COMPOSIO_SEARCH_WEB -d "$(jq -n --arg q "$Q" '{ query: $q }')"
  done
done
```

---

## Normalized JSON Shape

Use this shape for each competitor before building the final report:

```json
{
  "company": "{Company}",
  "pricing_url": "{URL}",
  "captured_at": "{ISO datetime}",
  "currency": "USD",
  "plans": [
    {
      "name": "Pro",
      "monthly_price": 49,
      "annual_price": 39,
      "billing_unit": "user/month",
      "seat_minimum": 1,
      "included_usage": ["10 projects", "100GB storage"],
      "feature_limits": ["basic analytics", "standard support"],
      "addons": ["extra usage billed separately"],
      "trial": "14 days, credit card not required",
      "cta": "Start free trial",
      "notes": "Annual price shown as monthly equivalent."
    }
  ],
  "enterprise": {
    "available": true,
    "cta": "Contact sales",
    "claims": ["SSO", "SOC 2", "custom limits"]
  },
  "source_quality": "direct_browser",
  "ambiguities": ["Usage overage listed in docs, not pricing page."]
}
```

---

## Output Format

```markdown
---
topic: {SPACE_OR_PRODUCT_CATEGORY}
competitors_analyzed: {N}
pricing_pages_checked: {N}
researched_at: {ISO datetime}
---

## Executive Summary

**Lowest entry price**: {company + plan + price}
**Most aggressive free tier**: {company + reason}
**Most enterprise-oriented**: {company + evidence}
**Notable pricing risk**: {where your pricing may look weak}

## Pricing Matrix

| Company | Plan | Monthly | Annual | Billing Unit | Trial / Free | Key Limits |
|---------|------|---------|--------|--------------|--------------|------------|
| {Company} | {Plan} | ${N} | ${N} | user/mo | 14-day trial | {limits} |

## Packaging Differences

| Feature / Limit | {Competitor A} | {Competitor B} | {Competitor C} |
|-----------------|----------------|----------------|----------------|
| Seats included | {value} | {value} | {value} |
| Usage cap | {value} | {value} | {value} |
| SSO | {plan} | {plan} | {plan} |

## Change Signals

- {Company}: {pricing or packaging change signal} — {source URL}
- {Company}: {new plan / removed plan / new add-on} — {source URL}

## Positioning Notes

- **Upgrade triggers**: {features competitors use to push paid plans}
- **Discounting pattern**: {annual savings, launch discounts, hidden promos}
- **Enterprise pattern**: {security/procurement claims and contact-sales usage}

## Source Notes

| Company | Source Quality | Ambiguities |
|---------|----------------|-------------|
| {Company} | direct browser | {anything unclear or hidden} |
```

---

## Guardrails

- Do not submit signup, checkout, account-creation, payment, demo-request, or
  contact-sales forms without explicit user approval.
- Do not bypass paywalls, logins, access controls, or robots-denied private
  areas. Use only public pages unless the user provides authorized access.
- Treat pricing claims from search snippets, third-party pages, and review sites
  as unverified until confirmed on the vendor website.
- Preserve source URLs for every factual pricing claim.
- Flag ambiguity instead of guessing when billing units, annual discounts, tax,
  or usage overages are unclear.

---

## Key Use Cases

1. **Pricing strategy** — benchmark your own packaging against competitors
2. **Sales enablement** — prepare competitor battlecards with current prices
3. **Product marketing** — identify plan gates and positioning language
4. **Finance planning** — track vendor pricing for procurement and renewals
5. **Competitive intelligence** — detect new plans, add-ons, and expansion levers
