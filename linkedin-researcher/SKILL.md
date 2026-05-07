---
name: linkedin-researcher
description: |
  Research people and companies on LinkedIn — role history, skills, recent
  activity, mutual connections (when logged in), and company signals like
  headcount growth and recent hires. Useful for pre-call prep, finding
  decision-makers, mapping an org chart, and personalizing outreach.
when_to_use: |
  "research {person} on LinkedIn", "LinkedIn profile for", "who leads X at Y",
  "find decision maker at", "background on {person}", "map the org chart for".
license: MIT
compatibility: Requires the `composio` CLI on PATH. Connect with `composio link browser_tool` (interactive cloud browser) and `composio link composio_search` (web search and URL fetch).
allowed-tools: Bash Agent
---

# LinkedIn Researcher

Research people and companies on LinkedIn. LinkedIn blocks local browsers and rate-limits aggressively — always run via the cloud browser, keep sessions short, and pause on CAPTCHAs rather than try to bypass them.

**Output directory**: `~/Desktop/{slug}_linkedin_{YYYY-MM-DD}/`

---

## What It Extracts

### Person Profile
- Current role and company
- Previous roles (last 3-5 positions with dates)
- Education
- Skills (top 10)
- Recent posts and activity
- Mutual connections (if logged in)
- Contact info (if visible)

### Company Page
- Headcount and recent growth
- Recent hires by department
- Key people (leadership, decision-makers)
- Recent company posts and announcements
- Job openings (signals team growth)

---

## Pipeline

### Step 1: Find the profile URL

Search Google first — it avoids LinkedIn's login wall on the search page:

```bash
composio execute COMPOSIO_SEARCH_WEB -d '{ "query": "site:linkedin.com/in {NAME} {COMPANY}" }'
# Pick the canonical /in/<slug> URL.
```

### Step 2: Profile

```bash
composio execute BROWSER_TOOL_CREATE_TASK -d '{
  "task":     "Capture the full profile: headline, current role and company, location, the last 5 roles with title/company/dates, education, top 10 skills, and the 5 most recent posts or activity items. Scroll until all of these sections are loaded. If a CAPTCHA or login wall appears, stop and report it — do not attempt to bypass.  Stop when: all sections captured OR auth wall encountered.",
  "startUrl": "{LINKEDIN_PROFILE_URL}"
}'
```

### Step 3: Company page (optional)

```bash
composio execute BROWSER_TOOL_CREATE_TASK -d '{
  "task":     "Capture: employee count, headcount-growth percentage if shown, recent company posts (last 5), and the People-also-viewed list. Stop if an auth wall appears.",
  "startUrl": "https://www.linkedin.com/company/{COMPANY_SLUG}"
}'
```

---

## Output Format

```markdown
---
name: {Full Name}
linkedin_url: {URL}
current_role: {Title} at {Company}
location: {City, Country}
researched_at: {ISO date}
---

## Current Role
- **Title**: {title}
- **Company**: {company}
- **Duration**: {start} – present ({N} years)

## Experience
| Role | Company | Duration |
|------|---------|----------|
| {title} | {company} | {start} – {end} |
| ... | ... | ... |

## Education
- {Degree}, {School} ({year})

## Top Skills
{skill1}, {skill2}, {skill3}, ...

## Recent Activity
- "{Post excerpt}" — {date}

## Outreach Notes
{Personalization hook based on research}
```

---

## Anti-Detection Rules

- Always run via the cloud browser (never a local profile).
- Keep tasks small — one profile per `BROWSER_TOOL_CREATE_TASK`. Don't batch dozens at once.
- Don't open more than ~20 profiles per session in total.
- On a CAPTCHA or "Let's do a quick security check": stop the task and report — do not attempt to bypass.

---

## Key Use Cases

1. **Pre-call research** — understand someone's background before a sales call
2. **Decision-maker mapping** — find the right person to contact at a company
3. **Org chart building** — map leadership at a target company
4. **Hiring signal detection** — recent job posts reveal company priorities
5. **Outreach personalization** — use recent posts/career moves as conversation openers