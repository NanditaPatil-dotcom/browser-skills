---
name: price-monitor
description: |
  Monitor product or SaaS plan prices across websites and detect changes,
  drops, restocks, and pricing-page edits. Supports single-URL snapshots,
  multi-site comparisons, and periodic checks with a price-history file
  for change detection.
when_to_use: |
  "monitor the price of", "track price changes on", "price drop alert for",
  "compare pricing across", "watch {SaaS} pricing", "when does {X} go on sale".
license: MIT
compatibility: Requires the `composio` CLI on PATH. Connect with `composio link browser_tool` (interactive cloud browser) and `composio link composio_search` (web search and URL fetch).
allowed-tools: Bash Agent
---

# Price Monitor

Monitor product prices across websites — detect changes, drops, and competitor pricing shifts. Supports one-time snapshots and periodic monitoring with change detection.

**Output directory**: `~/Desktop/price-monitor/` (persistent across runs)

---

## What It Tracks

### Product Pricing
- Current price
- Original/strikethrough price
- Discount percentage
- Stock status (in stock / out of stock / limited)
- Sale end date (if shown)

### SaaS Pricing Pages
- Plan names and prices
- Feature limits per plan
- Annual vs monthly toggle
- Hidden fees or add-ons

### Price History
- Price at each check (timestamped)
- % change from last check
- All-time high/low since monitoring started

---

## Pipeline

### Step 1: One-time price snapshot

Try fetch first — it's faster and cheaper, and works for static product pages:

```bash
composio execute COMPOSIO_SEARCH_FETCH_URL_CONTENT -d '{
  "urls": ["{PRODUCT_URL}"], "max_characters": 30000
}'
```

If the price isn't in the returned text (common on JS-heavy e-commerce), drive a browser:

```bash
composio execute BROWSER_TOOL_CREATE_TASK -d '{
  "task":     "Extract: current price, original / strike-through price, discount %, currency, stock status (in stock / out of stock / limited), sale-end date if shown, and seller. Return as a JSON object.",
  "startUrl": "{PRODUCT_URL}"
}'
```

### Step 2: SaaS pricing page

```bash
composio execute BROWSER_TOOL_CREATE_TASK -d '{
  "task":     "For every plan visible, capture: plan name, monthly price, annual price (toggle to annual if there is a billing toggle), feature limits (seats, projects, requests, etc.), and any add-ons. Return one JSON object per plan.",
  "startUrl": "{SAAS_URL}/pricing"
}'
```

### Step 3: Multi-site comparison

```bash
URLS=(
  "https://site1.com/product"
  "https://site2.com/product"
  "https://site3.com/product"
)

for URL in "${URLS[@]}"; do
  composio execute BROWSER_TOOL_CREATE_TASK -d "$(jq -n --arg u "$URL" '{
    url: $u,
    goal: "Extract: current price, original price, currency, stock status. Return JSON."
  }')" >> /tmp/all_prices.jsonl
done
```

### Step 4: Periodic Monitoring

Save snapshot to file and compare:
```bash
SNAPSHOT_FILE=~/Desktop/price-monitor/{slug}_history.json

# Read existing history
HISTORY=$(cat "$SNAPSHOT_FILE" 2>/dev/null || echo "[]")

# Add new price point
node -e "
  const history = JSON.parse('$HISTORY');
  const newEntry = { price: {CURRENT_PRICE}, timestamp: new Date().toISOString(), url: '{URL}' };
  history.push(newEntry);

  // Detect change
  if (history.length > 1) {
    const prev = history[history.length - 2].price;
    const curr = newEntry.price;
    const change = ((curr - prev) / prev * 100).toFixed(1);
    if (Math.abs(change) > 0) console.log('PRICE CHANGED:', change + '%', prev, '->', curr);
  }

  require('fs').writeFileSync('$SNAPSHOT_FILE', JSON.stringify(history, null, 2));
"
```

---

## Output Format

### Single Snapshot
```markdown
---
url: {URL}
product: {name}
checked_at: {ISO datetime}
---

## Current Price
- **Price**: \${N}
- **Original**: \${N} (if on sale)
- **Discount**: {N}% off
- **Stock**: In Stock / Out of Stock

## Price Context
- Sale ends: {date if shown}
- Shipping: {cost if shown}
- Seller: {marketplace seller if applicable}
```

### Price History (JSON)
```json
[
  { "price": 29.99, "timestamp": "2026-05-01T10:00:00Z", "stock": "in_stock" },
  { "price": 24.99, "timestamp": "2026-05-06T10:00:00Z", "stock": "in_stock", "change": "-16.7%" }
]
```

### SaaS Pricing Comparison
```markdown
## {Company} Pricing — {date}

| Plan | Monthly | Annual | Key Limits |
|------|---------|--------|------------|
| Free | \$0 | \$0 | {limits} |
| Pro | \${N}/mo | \${N}/yr | {limits} |
| Team | \${N}/mo | \${N}/yr | {limits} |
| Enterprise | Custom | Custom | Unlimited |

## Changes Since Last Check
- **{Plan}**: \${old} → \${new} ({change}%)
- New plan added: {name}
```

---

## Key Use Cases

1. **Competitive pricing** — track when competitors change their SaaS pricing
2. **Deal hunting** — monitor a product URL until price drops
3. **Restock alerts** — check stock status periodically
4. **Market research** — compare pricing across multiple vendors
5. **Procurement** — track software pricing for budget planning