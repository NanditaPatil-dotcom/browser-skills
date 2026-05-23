# competitor-pricing-tracker Examples

## Example: compare three SaaS competitors

User request:

```text
Compare pricing and packaging for Linear, Jira, and Height. Focus on team
plans, annual discounts, seat minimums, and enterprise CTAs.
```

Expected workflow:

1. Search for each company's pricing page and billing docs.
2. Fetch static pricing content with `COMPOSIO_SEARCH_FETCH_URL_CONTENT`.
3. Use `BROWSER_TOOL_CREATE_TASK` for toggles, feature matrices, accordions,
   and contact-sales sections.
4. Normalize each competitor into the JSON shape from `SKILL.md`.
5. Produce a pricing matrix, packaging comparison, and source notes.

## Example: monitor a known pricing page

User request:

```text
Track changes on https://example.com/pricing and tell me if any plan prices,
seat limits, or trial terms changed since last month.
```

Example payload:

```json
{
  "company": "Example",
  "pricing_url": "https://example.com/pricing",
  "watch_fields": [
    "plan_names",
    "monthly_price",
    "annual_price",
    "seat_minimum",
    "included_usage",
    "trial_terms",
    "enterprise_cta"
  ]
}
```

Expected output:

- Current normalized pricing JSON
- Human-readable pricing matrix
- Change list against the previous snapshot if available
- Ambiguities and source-quality notes

## Example: pricing battlecard

User request:

```text
Build a competitor pricing battlecard for AI customer support tools:
Intercom, Zendesk, Gorgias, and Freshdesk.
```

Expected output sections:

- Executive summary
- Entry-level paid plan comparison
- Free trial and free-plan comparison
- Feature gates that force upgrades
- Enterprise and security positioning
- Suggested pricing risks or opportunities

## Notes

- The skill should not submit signup, checkout, demo, or contact forms.
- If a page requires login or payment details to expose pricing, report that
  limitation instead of trying to bypass it.
- Preserve source URLs for every pricing claim.

