---
name: app-store-reviews
description: |
  Scrape App Store (iOS) and Google Play reviews for any app and extract pain
  points, praise, feature requests, churn signals, and competitor mentions.
  Returns a rating distribution, themed quote clusters, and a ranked feature
  request list grounded in real user language.
when_to_use: |
  "app store reviews for {app}", "google play reviews for {app}",
  "what do users say about {app}", "mobile app feedback on {app}",
  "iOS reviews for {app}", "play store analysis for {app}".
license: MIT
compatibility: Requires the `composio` CLI on PATH. Connect with `composio link browser_tool` (interactive cloud browser) and `composio link composio_search` (web search and URL fetch). The iTunes review feed is keyless.
allowed-tools: Bash Agent
---

# App Store Reviews

Scrape iOS App Store and Google Play reviews to extract pain points, praise, and feature requests from real mobile users. iTunes is API-only (no browser); Google Play needs an interactive browser session.

**Output directory**: `~/Desktop/{slug}_reviews_{YYYY-MM-DD}/`

---

## What It Extracts

### Review Data
- Rating (1-5 stars)
- Review title and body
- Username and date
- Version reviewed
- Developer response (if any)

### Aggregated Signals
- Rating distribution (1★ vs 5★ %)
- Most common words in negative reviews
- Feature requests extracted from reviews
- Competitor mentions ("switched from X", "better than Y")
- Churn signals ("cancelling", "uninstalling", "requesting refund")

---

## Data Sources

| Platform | Source | Method |
|----------|--------|--------|
| iOS App Store | iTunes RSS API | Free, no auth |
| Google Play | `BROWSER_TOOL_CREATE_TASK` on play.google.com | Cloud browser |
| App ratings | iTunes Search API | Free |

---

## Pipeline

### Step 1: Find App IDs

```bash
# iOS — search iTunes API
APP_NAME="{APP_NAME}"
curl -s "https://itunes.apple.com/search?term=${APP_NAME// /+}&entity=software&limit=5" \
  | node -e "
    const r = JSON.parse(require('fs').readFileSync('/dev/stdin','utf8'));
    r.results.forEach(a => console.log(a.trackId, a.trackName, a.averageUserRating));
  "

# iOS — get app details
APP_ID="{IOS_APP_ID}"
curl -s "https://itunes.apple.com/lookup?id=${APP_ID}" \
  | node -e "
    const r = JSON.parse(require('fs').readFileSync('/dev/stdin','utf8'));
    const a = r.results[0];
    console.log(a.trackName, a.averageUserRating, a.userRatingCount);
  "
```

### Step 2: Fetch App Store Reviews (RSS Feed)

```bash
# iOS App Store RSS — up to 500 most recent reviews
COUNTRY="us"
curl -s "https://itunes.apple.com/${COUNTRY}/rss/customerreviews/page=1/id=${APP_ID}/sortby=mostrecent/json" \
  | node -e "
    const d = JSON.parse(require('fs').readFileSync('/dev/stdin','utf8'));
    const reviews = d.feed?.entry || [];
    reviews.forEach(r => {
      if (!r.id || !r.content) return;
      console.log(JSON.stringify({
        rating: r['im:rating']?.label,
        title: r.title?.label,
        body: r.content?.label,
        author: r.author?.name?.label,
        date: r.updated?.label,
        version: r['im:version']?.label
      }));
    });
  "

# Get pages 2-5 (each page has ~50 reviews)
for PAGE in 2 3 4 5; do
  curl -s "https://itunes.apple.com/${COUNTRY}/rss/customerreviews/page=${PAGE}/id=${APP_ID}/sortby=mostrecent/json"
  sleep 1
done
```

### Step 3: Google Play Reviews

Goal-driven cloud browser task. The agent describes what to extract and the tool runs the multi-step flow (open page, sort, scroll to load more, capture):

```bash
composio execute BROWSER_TOOL_CREATE_TASK -d '{
  "task":     "Sort reviews by newest, then scroll until at least 100 reviews are loaded. For each review extract: rating (1-5), title, body, username, date, and version. Return as a JSON array. Stop when 100 are captured or the page stops loading more.  Stop when: 100 reviews captured OR no new reviews load for 3 consecutive scrolls.",
  "startUrl": "https://play.google.com/store/apps/details?id={PACKAGE_NAME}&showAllReviews=true"
}'

# Capture watch_task_id from the response, then poll:
composio execute BROWSER_TOOL_WATCH_TASK -d '{ "taskId": "<watch_task_id>" }'
# Read response.data.output once response.data.status == "finished" and is_success == true.
```

### Step 4: Analyze Reviews

Filter by rating:
- **1-2 star**: pain points, churn reasons, bugs
- **3 star**: mixed signals, missing features
- **4-5 star**: love signals, what to preserve

Extract:
```bash
# Keyword frequency analysis from reviews
node -e "
  const reviews = require('./reviews.json');
  const keywords = ['crash', 'slow', 'bug', 'love', 'great', 'expensive', 'switched', 'missing', 'wish', 'please add'];
  const counts = {};
  keywords.forEach(k => counts[k] = reviews.filter(r => r.body.toLowerCase().includes(k)).length);
  console.log(JSON.stringify(counts, null, 2));
"
```

---

## Output Format

```markdown
---
app_name: {Name}
ios_app_id: {ID}
play_package: {com.example.app}
ios_rating: {N}/5 ({N} ratings)
play_rating: {N}/5 ({N} ratings)
reviews_analyzed: {N}
researched_at: {ISO date}
---

## Rating Distribution
| Stars | iOS | Google Play |
|-------|-----|-------------|
| ★★★★★ | {N}% | {N}% |
| ★★★★☆ | {N}% | {N}% |
| ★★★☆☆ | {N}% | {N}% |
| ★★☆☆☆ | {N}% | {N}% |
| ★☆☆☆☆ | {N}% | {N}% |

## Top Pain Points (1-2★ Reviews)
- "{review excerpt}" — {date}, {platform}
- "{review excerpt}" — {date}, {platform}

## Top Praise (4-5★ Reviews)
- "{review excerpt}" — {date}, {platform}

## Feature Requests (from reviews)
| Feature | Mentions | Example Quote |
|---------|----------|---------------|
| {feature} | {N} | "..." |

## Competitor Mentions
- {App name}: mentioned {N} times (switched to/from)

## Churn Signals
- "Uninstalling because..." — {N} mentions
- "Cancelling subscription because..." — {N} mentions
```

---

## Key Use Cases

1. **PMF research** — what do users of competing apps actually want?
2. **Competitive intelligence** — where do competitor users get frustrated?
3. **Feature validation** — are users asking for what you're building?
4. **Churn analysis** — why do people uninstall or cancel?
5. **Copy research** — what language do real users use to describe the problem?