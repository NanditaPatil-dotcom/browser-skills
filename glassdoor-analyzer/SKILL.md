---
name: glassdoor-analyzer
description: |
  Scrape Glassdoor for company reviews, salary data, interview experiences,
  and culture signals. Returns ratings breakdown, top pro / con quotes,
  salary ranges by role and level, and an interview-process summary —
  useful for job research, sales personalization, and competitive talent
  analysis.
when_to_use: |
  "glassdoor reviews for", "what's it like to work at", "company culture at",
  "salary data for", "interview process at", "employee sentiment at".
license: MIT
compatibility: Requires the `composio` CLI on PATH. Connect with `composio link browser_tool` (interactive cloud browser) and `composio link composio_search` (web search and URL fetch).
allowed-tools: Bash Agent
---

# Glassdoor Analyzer

Scrape Glassdoor for company reviews, salary data, interview experiences, and culture signals. Glassdoor has aggressive bot detection — always run via the cloud browser.

**Output directory**: `~/Desktop/{slug}_glassdoor_{YYYY-MM-DD}/`

---

## What It Extracts

### Company Overview
- Overall rating (out of 5)
- Rating breakdown: Culture, Work/Life Balance, Management, Compensation, Career Growth
- CEO approval rating
- Recommend to a friend %
- Recent rating trend (improving/declining)

### Reviews
- Top positive reviews (what employees praise)
- Top critical reviews (pain points, churn reasons)
- Most mentioned pros and cons
- Management-specific complaints

### Salary Data
- Roles with salary ranges
- Compensation by seniority level
- Location-based salary variations

### Interview Data
- Difficulty rating
- Common interview questions
- Process length and stages
- Offer rate

---

## Pipeline

Run each step as a separate goal-driven task. Capture the watch-task id from `BROWSER_TOOL_CREATE_TASK` and poll `BROWSER_TOOL_WATCH_TASK` for the output. If a task hits a CAPTCHA or login wall, stop the task and split the goal smaller.

### Step 1: Find the company URL

```bash
composio execute COMPOSIO_SEARCH_WEB -d '{ "query": "site:glassdoor.com {COMPANY_NAME} reviews" }'
# Pick the URL of the form: glassdoor.com/Reviews/{company}-Reviews-{ID}.htm
```

### Step 2: Reviews — overall + recent

```bash
composio execute BROWSER_TOOL_CREATE_TASK -d '{
  "task":     "Capture: overall rating; the five sub-ratings (Culture & Values, Work/Life Balance, Senior Management, Compensation & Benefits, Career Opportunities); CEO approval %; Recommend-to-friend %. Then sort reviews by Most Recent and capture the latest 30 reviews — each as { title, rating, pros, cons, role, date }. Stop when 30 are captured or the page stops loading more.",
  "startUrl": "https://www.glassdoor.com/Reviews/{COMPANY_SLUG}-Reviews-{ID}.htm"
}'
```

### Step 3: Salaries

```bash
composio execute BROWSER_TOOL_CREATE_TASK -d '{
  "task":     "Extract every visible role with its level (junior/mid/senior/staff if shown), low/median/high salary, and total comp if shown. Return a JSON array.",
  "startUrl": "https://www.glassdoor.com/Salary/{COMPANY_SLUG}-Salaries-{ID}.htm"
}'
```

### Step 4: Interviews

```bash
composio execute BROWSER_TOOL_CREATE_TASK -d '{
  "task":     "Capture: difficulty rating, offer rate, average process length, and the 15 most upvoted interview questions with the role they apply to.",
  "startUrl": "https://www.glassdoor.com/Interview/{COMPANY_SLUG}-Interview-Questions-{ID}.htm"
}'
```

---

## Output Format

```markdown
---
company: {Name}
glassdoor_url: {URL}
overall_rating: {N}/5
total_reviews: {N}
researched_at: {ISO date}
---

## Ratings Breakdown
| Category | Score |
|----------|-------|
| Culture & Values | {N}/5 |
| Work/Life Balance | {N}/5 |
| Senior Management | {N}/5 |
| Compensation & Benefits | {N}/5 |
| Career Opportunities | {N}/5 |

**CEO Approval**: {N}%
**Recommend to Friend**: {N}%

## Top Pros (What Employees Love)
- "{Pro quote}" — {role}, {date}
- "{Pro quote}" — {role}, {date}

## Top Cons (Pain Points)
- "{Con quote}" — {role}, {date}
- "{Con quote}" — {role}, {date}

## Salary Ranges
| Role | Level | Salary Range |
|------|-------|-------------|
| Software Engineer | Senior | ${min}K – ${max}K |
| Product Manager | Mid | ${min}K – ${max}K |

## Interview Insights
- **Difficulty**: {Easy/Medium/Hard} ({N}% rating)
- **Process**: {description of stages}
- **Common Questions**:
  - "{question}"
  - "{question}"

## Culture Signals
| Signal | Finding |
|--------|---------|
| Management quality | {assessment} |
| Engineering culture | {assessment} |
| Remote/hybrid | {policy} |
| Growth opportunities | {assessment} |
```

---

## Key Use Cases

1. **Job research** — understand a company's culture before applying
2. **Sales intelligence** — find pain points to personalize outreach ("we saw your engineers rate management 2.1/5...")
3. **Competitive talent analysis** — is your competitor's team happy or about to leave?
4. **Hiring research** — understand what candidates care about at competitors
5. **Culture benchmarking** — compare ratings across companies in a space