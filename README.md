<h1 align="center">Browser Skills</h1>

<a href="https://dashboard.composio.dev/login?utm_source=Github&utm_medium=Github&utm_campaign=2026-05&utm_content=BrowserSkills">
<img width="1774" height="887" alt="Browser Skills Cover Image" src="https://github.com/user-attachments/assets/18e3bb49-1535-446d-a932-8953695f11f1" />
</a>
<p align="center">
  A curated collection of Browser Skills for automation — research, monitoring, and action-taking across the web.
</p>

<p align="center">
  <a href="https://awesome.re"><img src="https://awesome.re/badge.svg" alt="Awesome" /></a>
  <a href="https://makeapullrequest.com"><img src="https://img.shields.io/badge/PRs-welcome-brightgreen.svg?style=flat-square" alt="PRs Welcome" /></a>
  <a href="LICENSE"><img src="https://img.shields.io/badge/License-MIT-blue.svg?style=flat-square" alt="License: MIT" /></a>
</p>

A focused list of production-ready Skills for driving a real browser — scraping, mining communities, monitoring competitors, and taking actions on sites that require login, JavaScript, or anti-bot bypass. Skills follow the [open Agent Skills standard](https://github.com/anthropics/skills) and work across any compatible agent: Claude Code, Claude.ai, the Claude API, OpenAI Codex, Cursor, Gemini CLI, Antigravity, and Windsurf. Drop a skill into your agent's skills directory and it picks the skill up automatically when the task matches.

> **Want skills that do more than browse?** Skills can also send emails, create issues, post to Slack, and take actions across 1000+ apps. See [composiohq/awesome-claude-skills](https://github.com/composiohq/awesome-claude-skills).

---

## Quickstart

Skills here run on top of Composio's cloud browser — managed auth, anti-bot stealth, no local Chrome required.

**1. Install the Composio CLI**

```bash
# macOS / Linux
curl -fsSL https://install.composio.dev | sh

# or via npm
npm install -g composio

# verify
composio --version
```

Sign up for a free API key at [dashboard.composio.dev](https://dashboard.composio.dev) (you'll be prompted on first `composio login`).

**2. Connect the browser toolkits**

```bash
composio login
composio link browser_tool       # interactive cloud browser
composio link composio_search    # web search + URL fetch
composio connections list
```

**3. Discover and execute any tool from the CLI**

```bash
composio search "scrape product reviews" --toolkits browser_tool
composio execute BROWSER_TOOL_CREATE_TASK --get-schema
composio execute BROWSER_TOOL_CREATE_TASK -d '{
  "task":     "Extract the title, URL, and points of the top 10 stories. Stop when 10 are captured.",
  "startUrl": "https://news.ycombinator.com"
}'
```

---

## Contents

- [What Are Skills?](#what-are-skills)
- [Skills](#skills)
  - [Development & Code Tools](#development--code-tools)
  - [Data & Analysis](#data--analysis)
  - [Business & Marketing](#business--marketing)
  - [Productivity & Organization](#productivity--organization)
- [Getting Started](#getting-started)
- [Creating Skills](#creating-skills)
- [Contributing](#contributing)
- [Resources](#resources)
- [License](#license)

## What Are Skills?

Skills are reusable instruction packages that teach any compatible agent how to handle a specific class of tasks. Each skill is a folder containing a `SKILL.md` file with YAML frontmatter (name, description) and Markdown instructions, optionally bundled with scripts, references, and assets. Anthropic introduced the format in October 2025 and released it as an [open standard](https://github.com/anthropics/skills) in December 2025; the same skill folder works across Claude Code, Claude.ai, the Claude API, OpenAI Codex, Cursor, Gemini CLI, Antigravity, Windsurf, and any other agent that implements the standard.

Skills load progressively. At session start, the agent sees only each skill's name and description — roughly 100 tokens per skill. The full SKILL.md body (typically under 5,000 tokens) loads only when the agent decides the skill is relevant to the current task. Auxiliary files in `scripts/` and `references/` load on demand. This is what lets a single agent host hundreds of skills without bloating its context window.

Skills are not MCP servers and not tools. MCP defines how an agent connects to external systems — auth, transport, tool discovery. Tools are the individual functions an agent invokes. Skills define the workflow — what to do, in what order, with what guardrails — once the agent has the connections and tools it needs.

## Skills

### Development & Code Tools

- [browser](browser/) — Drive a real browser to navigate, click, fill forms, take screenshots, and extract content. Local Chrome or remote stealth. *By [browserbase/skills](https://github.com/browserbase/skills)*
- [fetch](fetch/) — Pull HTML, JSON, or page metadata from a URL without spinning up a full browser session. *By [browserbase/skills](https://github.com/browserbase/skills)*
- [search](search/) — Web search returning structured citations (title, URL, snippet, date). *By [browserbase/skills](https://github.com/browserbase/skills)*
- [github-tracker](github-tracker/) — Monitor GitHub repositories — stars, forks, issues, PRs, contributors, and release cadence. Produces a project-health snapshot.

### Data & Analysis

- [reddit-researcher](reddit-researcher/) — Mine subreddits for pain points, PMF signals, churn quotes, and competitor mentions. Ships a themed HTML report and CSV grounded in real user quotes.
- [hacker-news-digest](hacker-news-digest/) — Surface top HN threads on any topic and cluster top comments by sentiment (positive / skeptical / critical).
- [product-hunt-scout](product-hunt-scout/) — Track Product Hunt launches — new products, upvote trends, maker activity, and recurring user feedback in comments.
- [glassdoor-analyzer](glassdoor-analyzer/) — Scrape company reviews, salary data, and interview signals; useful for job research, sales personalization, and talent benchmarking.
- [app-store-reviews](app-store-reviews/) — Scrape App Store and Google Play reviews; cluster pain points, feature requests, and competitor mentions from real mobile users.
- [price-monitor](price-monitor/) — Monitor product or SaaS plan prices across sites; detect drops, restocks, and pricing-page changes with a price-history file.
- [competitor-pricing-tracker](competitor-pricing-tracker/) — Track competitor SaaS pricing, packaging, feature limits, add-ons, and pricing-page changes.
- [news-aggregator](news-aggregator/) — Collect and cluster news coverage on any topic across multiple outlets; identifies dominant narrative, dissenting takes, and missing angles.

### Business & Marketing

- [company-research](company-research/) — Discover and deeply research a company; score ICP fit; surface contacts and outreach hooks. *By [browserbase/skills](https://github.com/browserbase/skills)*
- [linkedin-researcher](linkedin-researcher/) — Research people and companies on LinkedIn — role history, recent activity, decision-maker mapping, and outreach personalization.
- [vc-research](vc-research/) — Research VCs and angels — portfolio, thesis, check sizes, partner backgrounds — with fit assessment and best-partner-to-contact.

### Productivity & Organization

- [job-tracker](job-tracker/) — Scrape job listings from company careers pages, ATS APIs (Greenhouse, Lever, Ashby), and LinkedIn Jobs into a structured pipeline.
- [job-applier](job-applier/) — Auto-fill job applications across Greenhouse, Lever, Ashby, Workday, and custom forms; pauses at the review screen for human approval before submit.
- [event-rsvp](event-rsvp/) — Auto-register / RSVP to events on Lu.ma, Eventbrite, Meetup, Hopin, and custom event sites; refuses paid events without explicit confirmation.

## Getting Started

1. Install the Composio CLI and connect the browser toolkits — see [Quickstart](#quickstart).
2. Drop any skill folder into your agent's skills directory (e.g. `.claude/skills/` for Claude Code, the equivalent for Codex / Cursor / Gemini CLI / Antigravity / Windsurf).
3. Ask the agent to do something the skill covers — it loads the skill automatically when the task matches the description.

Compose multiple skills for end-to-end pipelines:

```
reddit-researcher  →  company-research  →  linkedin-researcher
   pain points        target companies      decision makers
```

```
product-hunt-scout  →  app-store-reviews  →  news-aggregator
    new launches         user reception        media angle
```

```
job-tracker  →  glassdoor-analyzer  →  job-applier
  find roles      culture / salary     auto-fill + human review
```

## Creating Skills

Each skill is a directory with at minimum a `SKILL.md`:

```
my-skill/
├── SKILL.md          # required — frontmatter + instructions
├── scripts/          # optional — runnable scripts
├── EXAMPLES.md       # optional
└── references/       # optional — files the agent loads on demand
```

Frontmatter best practices: keep `description` user-intent focused (no implementation tool names), list trigger phrases under `when_to_use`, and pre-approve any tools the skill needs:

```yaml
---
name: my-skill
description: |
  One-paragraph statement of what the skill does and the kind of result it
  produces. Lead with the user's intent, not the tool used underneath.
when_to_use: |
  "trigger phrase one", "trigger phrase two", "another way a user might ask".
license: MIT
compatibility: Requires the `composio` CLI on PATH; connect with `composio link browser_tool`.
allowed-tools: Bash Agent
---
```

See Anthropic's [skills documentation](https://code.claude.com/docs/en/skills) for the full reference.

## Contributing

PRs welcome. To add a skill:

1. Create `your-skill-name/` at the repo root.
2. Write `SKILL.md` following the format above.
3. Add scripts, references, or examples as needed.
4. Add a one-line entry to the right category in this README — format `[skill-name](skill-name/) — brief description.`
5. Open a PR.

## Resources

- [Anthropic Skills documentation](https://code.claude.com/docs/en/skills)
- [Open Skills standard](https://github.com/anthropics/skills)
- [browserbase/skills](https://github.com/browserbase/skills) — upstream for `browser`, `fetch`, `search`, `company-research`
- [composiohq/awesome-claude-skills](https://github.com/composiohq/awesome-claude-skills) — broader Claude Skills directory across all app categories
- [Composio docs](https://docs.composio.dev) — `browser_tool` and `composio_search` toolkits

## License

MIT. Skills marked as upstream from [browserbase/skills](https://github.com/browserbase/skills) carry their own MIT license.
