---
name: product-hunt-scout
description: |
  Track Product Hunt launches — new products, upvote trends, maker activity,
  and user feedback in comments. Returns ranked launches by category with
  sentiment-tagged top comments, maker responses, and patterns across
  recent launches (recurring praise, recurring criticism).
when_to_use: |
  "product hunt launches in", "what's new on PH", "find PH launches in {topic}",
  "who launched {product}", "PH comments for", "product hunt research".
license: MIT
compatibility: Requires the `composio` CLI on PATH. Connect with `composio link browser_tool` (interactive cloud browser) and `composio link composio_search` (web search and URL fetch).
allowed-tools: Bash Agent
---

# Product Hunt Scout

Track Product Hunt launches, upvote trends, maker activity, and user comments. Extracts competitive intelligence and community feedback from PH's public content.

**Output directory**: `~/Desktop/{slug}_ph_{YYYY-MM-DD}/`

---

## What It Extracts

### Launch Data
- Product name, tagline, description
- Upvote count and rank (today / this week / this month)
- Maker names and backgrounds
- Launch date
- Category / topics
- Links (website, GitHub, App Store)

### Community Signals
- Top comments (positive and critical)
- Maker responses
- Hunter background (influential vs unknown)
- Follower-to-upvote ratio (organic vs paid)

### Category Trends
- Most upvoted products in a category over time
- Recurring pain points in comments
- Features users explicitly praise or request

---

## Data Sources

| Source | What | How |
|--------|------|-----|
| `producthunt.com/posts` | Today's launches | `BROWSER_TOOL_CREATE_TASK` |
| `producthunt.com/topics/{topic}` | Category products | `BROWSER_TOOL_CREATE_TASK` |
| `api.producthunt.com` | Structured data | direct HTTPS (OAuth for writes) |
| Web search for older launches | Find specific products | `COMPOSIO_SEARCH_WEB` |

---

## Pipeline

### Step 1: Category or daily feed

```bash
composio execute BROWSER_TOOL_CREATE_TASK -d '{
  "task":     "Capture the top 20 launches: name, tagline, upvote count, rank if shown, hunter, maker(s), launch date, topics, and the linked website URL. Return as a JSON array.",
  "startUrl": "https://www.producthunt.com/topics/{TOPIC}"
}'
```

For today's top: swap the URL for `https://www.producthunt.com`.

### Step 2: Discover older launches

```bash
composio execute COMPOSIO_SEARCH_WEB -d '{ "query": "site:producthunt.com {TOPIC} {YEAR}" }'
```

### Step 3: Individual product page (deep)

```bash
composio execute BROWSER_TOOL_CREATE_TASK -d '{
  "task":     "Capture: tagline, full description, upvote count, all maker names, hunter, launch date, all linked URLs, and the top 20 comments sorted by upvotes — each as { username, text, upvotes }, plus any maker responses. Scroll until 20 comments are visible.",
  "startUrl": "https://www.producthunt.com/posts/{PRODUCT_SLUG}"
}'
```

### Step 4: Maker research

```bash
composio execute BROWSER_TOOL_CREATE_TASK -d '{
  "task":     "Capture the maker'\''s previous launches (name, year, upvotes), their bio, and any company/team information visible.",
  "startUrl": "https://www.producthunt.com/@{MAKER_USERNAME}"
}'
```

---

## Output Format

```markdown
---
topic: {TOPIC}
total_products: {N}
date_range: {time period}
researched_at: {ISO date}
---

## Top Products

### 1. {Product Name}
**Tagline**: {tagline}
**Upvotes**: {N} | **Rank**: #{N} of the day/week
**URL**: https://producthunt.com/posts/{slug}
**Website**: {URL}
**Launched**: {date}
**Maker**: {name} (@{handle})
**Topics**: {topic1}, {topic2}

**Description**:
{product description}

**Top Comments**:
> "{comment}" — {username}, {upvotes}↑

> "{comment}" — {username}

**Maker Response**:
> "{maker quote}"

**Sentiment**: Positive | Mixed | Critical
**Key Praise**: {what users love}
**Key Criticism**: {what users complain about}

---

## Category Trends

| Signal | Finding |
|--------|---------|
| Most upvoted feature | {feature mentioned most} |
| Recurring pain point | {problem in comments} |
| Common alternatives mentioned | {tools compared to} |
| Launch timing | {what day/time gets traction} |

## Maker Landscape
| Maker | Products Launched | Notable Background |
|-------|------------------|--------------------|
| {name} | {N} | {company/background} |
```

---

## Key Use Cases

1. **Competitive monitoring** — track competitors launching on PH
2. **Market mapping** — what's been built in a category over the past year
3. **Launch research** — what makes a successful PH launch (comments, timing)
4. **Founder discovery** — find active builders in a space
5. **User feedback mining** — real user reactions to new products