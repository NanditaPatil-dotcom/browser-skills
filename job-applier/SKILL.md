---
name: job-applier
description: |
  Auto-fill job applications across Greenhouse, Lever, Ashby, Workday, and
  custom company forms — given a candidate profile JSON (name, contact,
  resume URL, work history, links) and a job posting URL, the skill drives
  a real cloud browser to fill every visible field, attach the resume, and
  pause for human confirmation before final submit. Companion to
  `job-tracker` (which finds the roles).
when_to_use: |
  "apply to {role} at {company}", "fill out the application at {url}",
  "submit my resume to", "apply to these job postings", "auto-apply to",
  "complete the Greenhouse application for".
license: MIT
compatibility: Requires the `composio` CLI on PATH. Connect with `composio link browser_tool` (interactive cloud browser) and `composio link composio_search` (web search and URL fetch).
allowed-tools: Bash Agent
---

# Job Applier

Drive a real browser through a job application end-to-end: parse the form, fill every visible field from a structured candidate profile, attach the resume, and **stop before the irreversible submit** so a human can review.

**Hard rules**:
- Never click final Submit / Send / Apply Now without explicit user confirmation.
- Never fabricate answers Claude doesn't have grounded data for (years of experience, salary expectation, work-auth status, etc.) — pause and ask the user.
- One application per task. Multiple applications = multiple `BROWSER_TOOL_CREATE_TASK` calls.

---

## Setup

```bash
composio login
composio link browser_tool
composio connections list
```

Maintain a candidate profile at `~/Desktop/job-applier/profile.json`:

```json
{
  "first_name": "Alex",
  "last_name":  "Doe",
  "email":      "alex@example.com",
  "phone":      "+1-555-0100",
  "location":   "San Francisco, CA, USA",
  "linkedin":   "https://www.linkedin.com/in/alexdoe",
  "github":     "https://github.com/alexdoe",
  "portfolio":  "https://alexdoe.dev",
  "resume_url": "https://alexdoe.dev/resume.pdf",
  "work_authorization": "US Citizen — no sponsorship needed",
  "years_experience":   8,
  "current_title":      "Senior Backend Engineer",
  "current_company":    "Acme Inc",
  "preferred_pronouns": "they/them",
  "willing_to_relocate": false,
  "notice_period":       "2 weeks",
  "salary_expectation":  "competitive",
  "how_did_you_hear":    "Referral",
  "cover_letter_template": "Dear hiring team at {COMPANY}, ..."
}
```

Resume must be a publicly fetchable HTTPS URL (the cloud browser cannot read your local filesystem).

---

## Pipeline

### Step 1: Inspect the form first

Don't fill blind — let the agent enumerate fields so the user sees what's being asked:

```bash
composio execute BROWSER_TOOL_CREATE_TASK -d '{
  "task":     "Open the application form. Enumerate every visible input field and return as a JSON array, each item: { label, type (text|email|phone|select|file|textarea|checkbox|radio|url), required (true/false), options (for select/radio) }. Do NOT fill anything yet. Stop after enumeration.",
  "startUrl": "{JOB_POSTING_URL}"
}'
```

Poll `BROWSER_TOOL_WATCH_TASK` and review the field list. If any field requires data not in `profile.json`, stop and ask the user.

### Step 2: Fill the form (no submit)

Once the user has confirmed which answers to use for any ambiguous fields, fill in a single goal-driven task:

```bash
composio execute BROWSER_TOOL_CREATE_TASK -d '{
  "task":     "Fill out the application form using the profile JSON below. Attach the resume by uploading from resume_url. For long-text fields like cover letter, tailor the cover_letter_template to mention the role title and company name visible on the page. CRITICAL: do NOT click Submit / Apply / Send. After filling every field, stop on the review screen and report back: which fields were filled, which were left blank, and any field whose dropdown options did not match the profile data. Profile: {PROFILE_JSON}",
  "startUrl": "{JOB_POSTING_URL}",
  "sessionId": "{previous_session_id_from_step_1}"
}'
```

Reusing `sessionId` keeps cookies (so login walls and CSRF tokens carry over). For ATS pages without auth, omit `sessionId`.

### Step 3: Show the user the live viewer

```bash
composio execute BROWSER_TOOL_GET_SESSION -d '{ "sessionId": "{session_id}" }'
# Share the returned liveUrl so the user can watch and confirm before submit.
```

### Step 4: Submit (only on explicit user confirmation)

```bash
composio execute BROWSER_TOOL_CREATE_TASK -d '{
  "task":     "Click the Submit / Apply button on the current review screen. Then capture the confirmation page text (e.g., \"Application received\", confirmation number) and return as JSON. Stop immediately if there is any further confirmation modal — do NOT auto-confirm.",
  "sessionId": "{session_id}"
}'
```

### Step 5: Log to pipeline

```bash
PIPELINE=~/Desktop/job-tracker/pipeline.json
node -e "
  const fs = require('fs');
  const p = fs.existsSync('$PIPELINE') ? JSON.parse(fs.readFileSync('$PIPELINE','utf8')) : [];
  p.push({
    company:    '{COMPANY}',
    role:       '{ROLE}',
    url:        '{JOB_POSTING_URL}',
    status:     'applied',
    applied_at:  new Date().toISOString(),
    confirmation: '{CONFIRMATION_NUMBER}'
  });
  fs.writeFileSync('$PIPELINE', JSON.stringify(p, null, 2));
"
```

---

## Per-ATS notes

| ATS | Notes |
|-----|-------|
| **Greenhouse** | Form is one long page. Resume upload is reliable. Often asks "How did you hear about us?" — pull from `profile.how_did_you_hear`. |
| **Lever** | Two-step (resume → form). Auto-fills from resume, then asks for confirmation. |
| **Ashby** | Multi-step wizard. Use `sessionId` continuity across `CREATE_TASK` calls per page. |
| **Workday** | Requires account creation. Pause on the account-creation screen and ask the user. |
| **Custom forms** | Enumerate first (Step 1), expect surprises, never assume field semantics. |

---

## Refuse / pause conditions

The agent must stop the task and report instead of guessing when:

- A required field has no matching key in `profile.json`.
- A salary or compensation field requires a number and `salary_expectation` is non-numeric.
- A demographic / EEO question asks for protected-class data — skip unless the user explicitly opts in.
- The page asks for a password (account creation) — pause.
- A CAPTCHA appears — pause, do not attempt to bypass.
- Workday / Workable / iCIMS multi-account flows — pause on first auth screen.

---

## Key use cases

1. **Targeted apply** — `job-tracker` finds 20 roles, you pick 5, this skill drafts the applications and stops at review.
2. **Mass apply (with review)** — auto-fill 10 Greenhouse forms in parallel, all paused at the review screen for human approval.
3. **Referral mode** — tweak `how_did_you_hear` to a referral name per application.
4. **Resume A/B testing** — swap `resume_url` per role to test which version converts better.
5. **Application audit trail** — every submission is logged to `~/Desktop/job-tracker/pipeline.json` with confirmation number.
