---
name: news-aggregator
description: |
  Collect and cluster news coverage on a topic across multiple outlets.
  Identifies the dominant narrative, dissenting takes, missing angles,
  and a timeline of how the story evolved. Returns articles grouped
  by angle with key quotes and a sentiment / outlet matrix.
when_to_use: |
  "news on", "media coverage of", "what's being written about",
  "press coverage of", "media monitoring for", "track coverage of {company}".
license: MIT
compatibility: Requires the `composio` CLI on PATH. Connect with `composio link browser_tool` (interactive cloud browser) and `composio link composio_search` (web search and URL fetch).
allowed-tools: Bash Agent
---

# News Aggregator

Collect and cluster news coverage on any topic from multiple sources. Extracts article angles, identifies narrative trends, and surfaces what different media outlets are emphasizing.

**No API keys required** for basic use (uses search + fetch).

**Output directory**: `~/Desktop/{slug}_news_{YYYY-MM-DD}/`

---

## What It Extracts

### Per Article
- Title, outlet, author, date
- Article summary (first 3-4 paragraphs)
- Key claims or angles
- Quotes from people mentioned
- Links to related coverage

### Across Articles
- Common narrative angles (what everyone is saying)
- Unique angles (contrarian or exclusive takes)
- Key people/companies quoted
- Sentiment trend (positive / negative / neutral)
- Timeline of how the story evolved

---

## Data Sources

| Source | Method | Notes |
|--------|--------|-------|
| Web / Google News | `COMPOSIO_SEARCH_NEWS` (recency-weighted) or `COMPOSIO_SEARCH_WEB` | Fast, broad |
| Direct outlets | `COMPOSIO_SEARCH_FETCH_URL_CONTENT` | Best quality, supports lists |
| RSS feeds | `COMPOSIO_SEARCH_FETCH_URL_CONTENT` on feed URL | Structured |
| Paywalled / JS-heavy | `BROWSER_TOOL_CREATE_TASK` | Stealth cloud browser |
| HN / Reddit | See those skills | Tech community |

---

## Pipeline

### Step 1: Discovery via search

```bash
TOPIC="{TOPIC}"
DATE_FILTER="after:$(date -d '7 days ago' +%Y-%m-%d)"

# Recency-weighted news
composio execute COMPOSIO_SEARCH_NEWS -d "$(jq -n --arg q "$TOPIC $DATE_FILTER" '{ query: $q }')"

# Specific angles (web search with constraints)
for ANGLE in "announcement" "analysis" "criticism OR concerns"; do
  composio execute COMPOSIO_SEARCH_WEB -d "$(jq -n --arg q "$TOPIC $ANGLE $DATE_FILTER" '{ query: $q }')"
done

# Specific outlets
for SITE in techcrunch.com theverge.com wired.com; do
  composio execute COMPOSIO_SEARCH_WEB -d "$(jq -n --arg q "site:$SITE $TOPIC" '{ query: $q }')"
done
```

De-duplicate `citations[*].url` across runs before fetching.

### Step 2: Fetch article content (batch)

```bash
# Pass up to ~10 URLs per call; iterate over statuses[] for partial failures
composio execute COMPOSIO_SEARCH_FETCH_URL_CONTENT -d "$(jq -n \
  --argjson urls "$(printf '%s\n' "${ARTICLE_URLS[@]}" | jq -R . | jq -s .)" \
  '{ urls: $urls, max_characters: 12000 }')"
```

### Step 3: Paywalled or JS-heavy outlets

When fetch returns empty or gated content:

```bash
composio execute BROWSER_TOOL_CREATE_TASK -d '{
  "task":     "Capture: headline, byline, publication date, and the first 5 body paragraphs of the article. Stop if a hard paywall blocks access — do not attempt to bypass.",
  "startUrl": "{PAYWALLED_URL}"
}'
```

### Step 4: Cluster by Angle

Group articles by narrative:
```
Angle A: "Company does X, here's why it matters"
Angle B: "Critics say X has problems"
Angle C: "Industry reactions to X"
Angle D: "What X means for the future of Y"
```

Identify:
- Dominant narrative (what most articles say)
- Dissenting views (what 1-2 outlets push back on)
- Missing angles (what nobody is covering)

---

## Output Format

```markdown
---
topic: {TOPIC}
articles_analyzed: {N}
date_range: {start} – {end}
outlets_covered: {N}
researched_at: {ISO date}
---

## Coverage Summary

**Dominant narrative**: {what most articles say}
**Dissenting view**: {contrarian take}
**Missing angle**: {what nobody is covering}

## Articles by Angle

### Angle: {Angle Name}

#### {Article Title}
**Outlet**: {Name} | **Author**: {Name} | **Date**: {date}
**URL**: {URL}

**Summary**: {2-3 sentence summary in own words}

**Key Quote**: "{quote from article}"

**Unique claim**: {what this article says that others don't}

---

## Key People Quoted
| Person | Role | Quoted In |
|--------|------|-----------|
| {name} | {title} | {N} articles |

## Timeline
| Date | Event | Outlet |
|------|-------|--------|
| {date} | {headline} | {outlet} |

## Sentiment Distribution
- Positive: {N}% of coverage
- Neutral: {N}%
- Negative/Critical: {N}%

## Outlets Coverage Matrix
| Outlet | Articles | Angle | Sentiment |
|--------|---------|-------|-----------|
| TechCrunch | {N} | {angle} | {sentiment} |
| The Verge | {N} | {angle} | {sentiment} |
```

---

## Key Use Cases

1. **Media monitoring** — track how your company/product is covered
2. **Competitive PR** — what's being written about your competitors
3. **Topic research** — understand a fast-moving story from multiple angles
4. **Journalist mapping** — who covers your beat and what angles they take
5. **Crisis monitoring** — track negative coverage in real time