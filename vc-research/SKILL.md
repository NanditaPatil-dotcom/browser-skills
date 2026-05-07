---
name: vc-research
description: |
  Research VCs and angel investors — portfolio companies, investment thesis,
  check sizes, stage focus, and partner backgrounds. Returns a targeted
  investor list with fit assessment, warm-intro paths when discoverable,
  and the best partner to contact for each firm.
when_to_use: |
  "find investors for {space}", "who funds {company}", "investor list for",
  "which VCs invest in {space}", "fundraising research", "find angels for",
  "map the funding landscape for".
license: MIT
compatibility: Requires the `composio` CLI on PATH. Connect with `composio link browser_tool` (interactive cloud browser) and `composio link composio_search` (web search and URL fetch).
allowed-tools: Bash Agent
---

# VC Research

Research venture capital firms, investors, and funding landscapes. Extracts portfolio companies, thesis, check sizes, stage focus, and partner contact details.

**Output directory**: `~/Desktop/{slug}_vc_research_{YYYY-MM-DD}/`

---

## What It Extracts

### VC Firm Profile
- Investment thesis and focus areas
- Stage focus (pre-seed, seed, Series A, etc.)
- Typical check size
- Geographic focus
- Portfolio companies (full list)
- Active partners + their focus areas
- Recent investments (signals current interest)
- Portfolio exits

### Individual Investor Profile
- Current firm + role
- Past investments (personal portfolio)
- Thesis and content (tweets, blog posts)
- LinkedIn/Twitter presence
- Best contact method

---

## Data Sources

| Source | What to find | How |
|--------|-------------|-----|
| `crunchbase.com` | Funding rounds, portfolio | `COMPOSIO_SEARCH_FETCH_URL_CONTENT`; fall back to browser |
| `pitchbook.com` | Detailed fund data | `BROWSER_TOOL_CREATE_TASK` (JS-rendered, gated) |
| `{firm}.com` | Thesis, team, portfolio | fetch first, browser if JS-heavy |
| `signal.nfx.com` | NFX portfolio signals | `BROWSER_TOOL_CREATE_TASK` |
| Twitter/X | Partner activity + thesis | `BROWSER_TOOL_CREATE_TASK` |
| LinkedIn | Partner backgrounds | use `linkedin-researcher` |

---

## Pipeline

### Step 1: Discover relevant VCs

```bash
for Q in \
  "venture capital {SPACE} seed {LOCATION}" \
  "who invests in {SPACE} startups" \
  "{COMPETITOR} investors funding"; do
  composio execute COMPOSIO_SEARCH_WEB -d "$(jq -n --arg q "$Q" '{ query: $q }')"
done
```

### Step 2: Research each firm

```bash
# Try fetch for the firm website + portfolio page
composio execute COMPOSIO_SEARCH_FETCH_URL_CONTENT -d '{
  "urls": ["{VC_WEBSITE}", "{VC_WEBSITE}/portfolio"], "max_characters": 25000
}'

# If JS-rendered or blocked, switch to the browser
composio execute BROWSER_TOOL_CREATE_TASK -d '{
  "task":     "Extract every portfolio company with: name, stage if shown, year of investment, and one-line description. Return a JSON array.",
  "startUrl": "{VC_WEBSITE}/portfolio"
}'
```

### Step 3: Find partner contacts

```bash
composio execute COMPOSIO_SEARCH_WEB -d '{ "query": "{FIRM_NAME} partner email contact" }'

composio execute BROWSER_TOOL_CREATE_TASK -d '{
  "task":     "For each partner: capture name, role, focus areas, Twitter handle, and any visible email or contact CTA.",
  "startUrl": "{VC_WEBSITE}/team"
}'
```

### Step 4: Recent activity

```bash
for Q in \
  "{FIRM_NAME} investment 2025 2026" \
  "{PARTNER_NAME} investment thesis blog"; do
  composio execute COMPOSIO_SEARCH_WEB -d "$(jq -n --arg q "$Q" '{ query: $q }')"
done
```

---

## Output Format

```markdown
---
firm_name: {Name}
website: {URL}
crunchbase: {URL}
stage_focus: pre-seed, seed
check_size: $250K - $2M
thesis: {1-2 sentence description}
geo_focus: US, Europe
researched_at: {ISO date}
---

## Thesis
{What they invest in, why, who for}

## Portfolio (Recent)
| Company | Stage | Year | Space |
|---------|-------|------|-------|
| {name} | Seed | 2025 | {space} |

## Partners
| Name | Focus | Contact |
|------|-------|---------|
| {name} | {focus} | {email/twitter} |

## Recent Investments (Signal)
- {Company} — {date} — {why relevant}

## Fit Assessment
**Why relevant**: {connection to your startup}
**Best partner to contact**: {name + reason}
**Warm intro path**: {mutual connection if known}
```

---

## Key Use Cases

1. **Fundraising prep** — build a targeted investor list before raising
2. **Competitive intelligence** — understand who funds your competitors
3. **Landscape mapping** — who are the key investors in a space
4. **Warm intro finding** — identify mutual connections for introductions
5. **Thesis alignment** — filter VCs whose thesis matches your startup