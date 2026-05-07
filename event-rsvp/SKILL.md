---
name: event-rsvp
description: |
  Auto-register or RSVP to events on Lu.ma, Eventbrite, Meetup, Hopin, and
  custom event sites — given an event URL and an attendee profile, the
  skill drives a real cloud browser to fill the registration form, answer
  screener questions from a saved profile, and stop before final submit
  for human confirmation. Tracks RSVPs to a local pipeline. Useful for
  hackathons, meetups, conferences, and product launches.
when_to_use: |
  "RSVP to {event url}", "register me for {event}", "sign up for the
  hackathon at", "register for this meetup", "fill out the conference form",
  "auto-RSVP to these Lu.ma events".
license: MIT
compatibility: Requires the `composio` CLI on PATH. Connect with `composio link browser_tool` (interactive cloud browser) and `composio link composio_search` (web search and URL fetch).
allowed-tools: Bash Agent
---

# Event RSVP

Drive a real browser through event registration end-to-end: parse the form, fill from a saved attendee profile, answer screener questions (T-shirt size, dietary needs, role, company), and **stop before submit** for human confirmation.

**Hard rules**:
- Never click final RSVP / Register / Confirm without explicit user confirmation.
- Never invent answers (T-shirt size, dietary, GitHub URL, etc.) — if a field doesn't map to the profile, pause and ask.
- One event per task.
- Refuse to register the user for events that charge money without explicit confirmation of the price.

---

## Setup

```bash
composio login
composio link browser_tool
composio connections list
```

Maintain an attendee profile at `~/Desktop/event-rsvp/profile.json`:

```json
{
  "first_name": "Alex",
  "last_name":  "Doe",
  "email":      "alex@example.com",
  "phone":      "+1-555-0100",
  "company":    "Acme Inc",
  "role":       "Senior Backend Engineer",
  "linkedin":   "https://www.linkedin.com/in/alexdoe",
  "github":     "https://github.com/alexdoe",
  "twitter":    "@alexdoe",
  "city":       "San Francisco",
  "country":    "USA",
  "tshirt":     "M",
  "dietary":    "no restrictions",
  "pronouns":   "they/them",
  "how_did_you_hear":      "Twitter",
  "what_brings_you":       "Curious about Composio's roadmap and meeting other builders.",
  "willing_to_be_recorded": true,
  "agree_to_code_of_conduct": true,
  "max_event_price_usd":    0
}
```

`max_event_price_usd: 0` means refuse paid events automatically. Set it higher (with explicit user confirmation per session) to allow paid registrations.

---

## Pipeline

### Step 1: Inspect the event + form

```bash
composio execute BROWSER_TOOL_CREATE_TASK -d '{
  "task":     "Capture: event title, date and time, venue (or \"online\"), host/organizer, listed price (or \"free\"), and a short description. Then enumerate every visible registration field as a JSON array of { label, type, required, options }. Do NOT fill or click Register yet. Stop after enumeration.",
  "startUrl": "{EVENT_URL}"
}'
```

Poll `BROWSER_TOOL_WATCH_TASK`. Review:
- Price > `profile.max_event_price_usd` → stop and report to user.
- Required field with no profile mapping → stop and ask.
- Date conflicts → flag (the user can sanity-check).

### Step 2: Fill the form (no submit)

```bash
composio execute BROWSER_TOOL_CREATE_TASK -d '{
  "task":     "Fill the registration form using the attendee profile JSON below. CRITICAL: do NOT click Register / Confirm / RSVP. After filling every field, stop on the confirmation screen and report: which fields were filled, which were left blank, and any required field whose options did not match the profile. Profile: {PROFILE_JSON}",
  "sessionId": "{session_id_from_step_1}"
}'
```

### Step 3: Show the user the live viewer

```bash
composio execute BROWSER_TOOL_GET_SESSION -d '{ "sessionId": "{session_id}" }'
# Share liveUrl so the user can watch + confirm.
```

### Step 4: Submit on explicit confirmation

```bash
composio execute BROWSER_TOOL_CREATE_TASK -d '{
  "task":     "Click the final Register / RSVP / Confirm button. Capture the confirmation page text and any confirmation number, ticket number, or QR code reference. Return as JSON. If a payment screen appears, STOP — do not enter card details.",
  "sessionId": "{session_id}"
}'
```

### Step 5: Log to local pipeline

```bash
LOG=~/Desktop/event-rsvp/registrations.json
node -e "
  const fs = require('fs');
  const p = fs.existsSync('$LOG') ? JSON.parse(fs.readFileSync('$LOG','utf8')) : [];
  p.push({
    event:       '{EVENT_TITLE}',
    url:         '{EVENT_URL}',
    date:        '{EVENT_DATE}',
    venue:       '{VENUE}',
    status:      'registered',
    registered_at: new Date().toISOString(),
    confirmation: '{CONFIRMATION}'
  });
  fs.writeFileSync('$LOG', JSON.stringify(p, null, 2));
"
```

Optional: emit an `.ics` file or push to Google Calendar via a separate skill.

---

## Per-platform notes

| Platform | Notes |
|----------|-------|
| **Lu.ma** | Often one-step. May require login via email magic link — pause if so. Some hosts auto-approve, others require approval; report which. |
| **Eventbrite** | Multi-step (tickets → details → review). Pause on payment if price > 0. |
| **Meetup** | Requires Meetup account. Pause on login screen. |
| **Hopin / Hopin Sessions** | Often requires account creation — pause. |
| **Custom (Webflow / Typeform / Tally)** | Enumerate first (Step 1), expect unusual field semantics. |

---

## Refuse / pause conditions

- Listed price exceeds `profile.max_event_price_usd`.
- Account creation required (passwords, magic links).
- A field asks for credit-card details.
- A required field has no profile mapping.
- The event title or description suggests something the user didn't ask for (e.g., a different city's chapter).
- A CAPTCHA appears.

---

## Batch RSVP

For a list of event URLs (e.g., a hackathon week's worth of side events):

```bash
jq -r '.events[]' events.json | while read -r URL; do
  echo "→ inspecting $URL"
  composio execute BROWSER_TOOL_CREATE_TASK -d "$(jq -n --arg u "$URL" '{
    task: "Capture event title, date, price, and enumerate registration fields. Stop after enumeration.",
    startUrl: $u
  }')"
  # Review the output, then run Step 2 + Step 4 per event after user approval.
done
```

---

## Key use cases

1. **Hackathon week** — RSVP to 10 side events in one sitting (with one human review per event).
2. **Conference circuit** — register for every relevant track/workshop at a multi-day conference.
3. **Local meetup tracking** — auto-RSVP to a recurring meetup series; log attendance.
4. **Product launches** — register for every launch event in a category.
5. **Founder office hours** — sign up to office-hours slots that fill fast.
