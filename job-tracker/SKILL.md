---
name: job-tracker
description: |
  Scrape job listings from company careers pages, ATS APIs (Greenhouse,
  Lever, Ashby), and LinkedIn Jobs; normalize roles into a structured
  pipeline with salary ranges, required skills, and hiring signals.
  Use to monitor a company's hiring, track personal applications, or
  surface what teams competitors are growing.
when_to_use: |
  "find jobs at", "open roles at", "track job applications", "hiring at",
  "who is hiring for", "scrape job postings", "job search pipeline".
license: MIT
compatibility: Requires the `composio` CLI on PATH. Connect with `composio link browser_tool` (interactive cloud browser) and `composio link composio_search` (web search and URL fetch).
allowed-tools: Bash Agent
---

# Job Tracker

Scrape job listings from multiple sources, normalize into a structured format, and maintain a searchable job pipeline. Extracts salary ranges, requirements, hiring signals, and application status.

**Output directory**: `~/Desktop/job-tracker/` (persistent pipeline)

---

## What It Extracts

### Per Job Listing
- Job title and seniority level
- Company name and size
- Location / remote policy
- Salary range (if listed)
- Required skills and years of experience
- Nice-to-have skills
- Job description (summarized)
- Application URL
- Posted date / closing date
- Hiring manager (if findable)

### Hiring Signals (per company)
- Number of open roles → growth signal
- Roles by department → which teams are expanding
- Role changes over time → track what a company is building

---

## Data Sources

| Source | Method | Notes |
|--------|--------|-------|
| Company careers pages | `COMPOSIO_SEARCH_FETCH_URL_CONTENT`, fallback to `BROWSER_TOOL_CREATE_TASK` | Most reliable |
| LinkedIn Jobs | `BROWSER_TOOL_CREATE_TASK` | Login wall — pause on auth |
| Greenhouse / Lever / Ashby | direct API (no browser needed) | Clean JSON |
| Indeed / Glassdoor | `BROWSER_TOOL_CREATE_TASK` | Anti-bot — cloud browser |
| Y Combinator jobs | `COMPOSIO_SEARCH_FETCH_URL_CONTENT` | `workatastartup.com` |

---

## Pipeline

### Step 1: Find Job Listings at a Company

```bash
# Check ATS directly (no browser needed — clean JSON)
# Greenhouse
curl -s "https://boards-api.greenhouse.io/v1/boards/{COMPANY_SLUG}/jobs" \
  | node -e "
    const d = JSON.parse(require('fs').readFileSync('/dev/stdin','utf8'));
    d.jobs.forEach(j => console.log(j.id, j.title, j.location?.name));
  "

# Lever
curl -s "https://api.lever.co/v0/postings/{COMPANY_SLUG}?mode=json" \
  | node -e "
    const jobs = JSON.parse(require('fs').readFileSync('/dev/stdin','utf8'));
    jobs.forEach(j => console.log(j.text, j.categories?.location, j.hostedUrl));
  "

# Ashby
curl -s "https://jobs.ashbyhq.com/api/non-user-graphql" \
  -H "Content-Type: application/json" \
  -d '{"operationName":"ApiJobBoardWithTeams","variables":{"organizationHostedJobsPageName":"{SLUG}"},"query":"query ApiJobBoardWithTeams($organizationHostedJobsPageName: String!) { jobBoard: jobBoardWithTeams(organizationHostedJobsPageName: $organizationHostedJobsPageName) { jobPostings { id title departmentName locationName employmentType } } }"}'
```

### Step 2: Company careers page (fallback when no ATS API)

Try fetch first; fall back to the browser if the page is JS-rendered:

```bash
composio execute COMPOSIO_SEARCH_FETCH_URL_CONTENT -d '{
  "urls": ["{COMPANY_URL}/careers"], "max_characters": 30000
}'

# Fallback
composio execute BROWSER_TOOL_CREATE_TASK -d '{
  "task":     "Capture every visible role: title, team/department, location, employment type, and the apply URL. Return a JSON array.",
  "startUrl": "{COMPANY_URL}/careers"
}'
```

### Step 3: Extract job details

```bash
composio execute BROWSER_TOOL_CREATE_TASK -d '{
  "task":     "Extract: title, seniority, location / remote policy, salary range (low/high/currency) if shown, required skills with years, nice-to-have skills, and the full tech stack mentioned. Return as JSON.",
  "startUrl": "{JOB_POSTING_URL}"
}'
```

### Step 4: LinkedIn Jobs

```bash
composio execute BROWSER_TOOL_CREATE_TASK -d '{
  "task":     "Capture the first 25 results: title, company, location, posted date, and apply URL. Stop on any auth wall — do not bypass.",
  "startUrl": "https://www.linkedin.com/jobs/search/?keywords={ROLE}&f_C={COMPANY_ID}"
}'
```

### Step 5: Track Application Status

Maintain `~/Desktop/job-tracker/pipeline.json`:
```bash
node -e "
  const fs = require('fs');
  const pipeline = JSON.parse(fs.readFileSync('pipeline.json', 'utf8') || '[]');
  pipeline.push({
    company: '{COMPANY}',
    role: '{ROLE}',
    url: '{URL}',
    salary: '{SALARY}',
    status: 'applied',   // applied | screen | interview | offer | rejected
    applied_at: new Date().toISOString(),
    notes: '{NOTES}'
  });
  fs.writeFileSync('pipeline.json', JSON.stringify(pipeline, null, 2));
"
```

---

## Output Format

### Job Listing (`raw/{company}-{role}.md`)
```markdown
---
company: {Name}
role: {Title}
seniority: {IC1 / IC2 / Senior / Staff / Principal}
location: {City / Remote / Hybrid}
salary_min: {N}
salary_max: {N}
ats: {greenhouse / lever / ashby / custom}
url: {application URL}
posted: {date}
status: {not_applied / applied / interviewing / offer / rejected}
---

## Summary
{2-3 sentence summary of the role}

## Required Skills
- {skill} ({years} years)
- {skill}

## Nice to Have
- {skill}
- {skill}

## Stack Mentioned
{tech1}, {tech2}, {tech3}

## Red Flags / Green Flags
- ✅ {positive signal}
- 🚩 {concern}
```

### Pipeline Dashboard (`pipeline.json`)
```json
[
  {
    "company": "Acme",
    "role": "Senior Engineer",
    "salary": "$180K-$220K",
    "status": "interview",
    "applied_at": "2026-05-01",
    "next_step": "Technical round May 10"
  }
]
```

---

## Key Use Cases

1. **Job search pipeline** — track all applications in one place
2. **Hiring signal detection** — monitor a company's open roles over time
3. **Competitive intelligence** — what is competitor X hiring for?
4. **Salary research** — aggregate salary ranges across similar roles
5. **Recruiter research** — find who's hiring in your target companies