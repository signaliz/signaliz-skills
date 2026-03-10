---
name: signaliz-company-signals
description: >
  Enrich companies with business signals (hiring, funding, tech stack, news,
  leadership changes, partnerships) using Signaliz MCP. Use this skill whenever
  someone asks to: enrich companies, find company signals, research company
  activity, discover hiring trends, check funding, detect buying signals, or
  run signal enrichment on a list of domains. Trigger on phrases like "enrich
  these companies", "find signals for", "what are these companies up to",
  "company enrichment", "buying signals", "enrich this list", "find trigger
  events", "batch company research", or any request to discover business
  activity across companies. Also trigger when someone uploads a CSV of domains
  and wants to know what's happening at those companies. Always use this skill
  for batch signal enrichment — even without mentioning Signaliz by name. For
  deep single-company research, consider company-intel-report instead.
---

# Company Signal Enrichment via Signaliz

Enrich a batch of companies with structured business signals — hiring trends,
funding activity, product launches, partnerships, leadership changes,
expansions, acquisitions, and more — using Signaliz's signal detection engine.
Accepts input from a CSV upload or directly from Claude.

---

## Before You Start

**Required:** The Signaliz MCP must be connected. If tools like
`enrich_company_signals` or `execute_primitive` aren't responding, tell the
user to disconnect and reconnect the Signaliz MCP in their Claude settings
(Supabase edge function cold starts can cause all tools to fail at once — a
reconnect fixes it).

**Tool loading:** Before making any Signaliz MCP calls, use `tool_search` to
load the correct tool definitions. Do not guess parameter names.

---

## Input Modes

### Mode A: CSV Upload

The user uploads a CSV containing company data. Inspect it and find the
domain column:

| Required Field    | Common CSV Variants                                        |
|-------------------|------------------------------------------------------------|
| `company_domain`  | Domain, Website, website, company_url, domain, URL, Site   |

Also helpful (not required):
| Field             | Common Variants                                            |
|-------------------|------------------------------------------------------------|
| `company_name`    | Company, Company Name, Name, Organization, Account         |

**If the CSV has `company_name` but no domain:** Attempt to derive domains
by searching `"[Company Name] official website"` for each. If you can't find
a domain confidently, include the record with `company_name` only (Signaliz
accepts either).

**Pre-processing:**
1. Normalize domains: strip `https://`, `http://`, `www.`, trailing slashes
2. Lowercase all domains
3. Deduplicate by domain
4. Remove empty rows

Report what you found:
```
Found 75 companies with domain column "Website"
Removed 3 duplicates, 2 empty rows
70 companies ready for signal enrichment
```

### Mode B: Inline / Conversational Input

The user provides companies directly — a list of domains, a list of company
names, or asks Claude to research companies first (e.g., "find signals for
the top 20 AI startups"). In this case:

1. Collect or research the company domains
2. Build the input list and confirm with the user before proceeding

---

## Execution: Enriching Company Signals

### Choosing the Right Tool

| Batch Size | Tool                                 | Pattern              |
|------------|--------------------------------------|----------------------|
| 1-25       | `execute_primitive` (capability: `enrich_company_signals`) | Instant/near-instant |
| 1-5000     | `Signaliz:enrich_company_signals`    | Async job + polling  |

### Small Batches (1-25 companies): `execute_primitive`

```json
{
  "capability_id": "enrich_company_signals",
  "input_data": [
    {"company_domain": "stripe.com"},
    {"company_domain": "notion.so"},
    {"company_domain": "linear.app"}
  ]
}
```

Optional config for deeper research:
```json
{
  "capability_id": "enrich_company_signals",
  "input_data": [...],
  "config": {
    "enable_deep_search": true,
    "signal_types": ["hiring", "funding", "product_launch", "leadership_change"],
    "research_prompt": "Focus on signals relevant to a GTM SaaS buyer"
  }
}
```

**If the CSV column isn't named `company_domain`**, use `field_mappings`:
```json
{
  "capability_id": "enrich_company_signals",
  "input_data": [...],
  "field_mappings": {
    "Website": "company_domain"
  }
}
```

### Large Batches (26+ companies): `Signaliz:enrich_company_signals`

Use the dedicated batch tool:

```json
{
  "companies": [
    {"domain": "stripe.com"},
    {"domain": "notion.so"},
    ...
  ],
  "config": {
    "enable_deep_search": true,
    "signal_types": ["hiring", "funding", "product_launch"]
  }
}
```

This returns a `job_id` immediately. Poll for results.

### Polling for Results

After getting a `job_id`, poll with `check_job_status`:

1. **First poll** — wait 10-15 seconds (signal enrichment takes longer than
   email verification), then call:
   ```json
   {"job_id": "<job_id>", "_poll_count": 1, "page_size": 500}
   ```
2. **Check status** — if `processing` or `queued`, wait the number of seconds
   indicated by `next_poll_after_seconds`
3. **Repeat** — increment `_poll_count` each call for proper backoff
4. **When `completed`** — results are in the response. Paginate if needed
5. **For deep search** — jobs may take 30-90 seconds. Set expectations with
   the user: "Deep search enrichment is running — this usually takes 30-60
   seconds for [N] companies."

---

## Available Signal Types

When the user has specific interests, filter with `signal_types`:

| Signal Type         | What It Finds                                              |
|---------------------|------------------------------------------------------------|
| `hiring`            | Open roles, headcount growth, new departments              |
| `funding`           | Rounds raised, investors, valuations                       |
| `product_launch`    | New products, features, major releases                     |
| `partnership`       | Strategic partnerships, integrations, alliances            |
| `leadership_change` | New C-suite hires, executive departures, reorgs            |
| `expansion`         | New offices, markets, geographies                          |
| `acquisition`       | M&A activity (acquiring or being acquired)                 |
| `award`             | Industry awards, recognition, rankings                     |
| `regulatory`        | Compliance changes, regulatory actions                     |
| `earnings`          | Revenue milestones, financial performance                  |

If the user doesn't specify, omit `signal_types` to get all available signals.

### Custom Research Prompts

For targeted signal discovery, use `research_prompt` in config:

```json
"config": {
  "enable_deep_search": true,
  "research_prompt": "Find signals that indicate this company may be evaluating new data quality or enrichment tools"
}
```

Good research prompts for GTM workflows:
- "Focus on pain signals: hiring for roles that suggest they're building in-house what we sell"
- "Find signals indicating budget availability: recent funding, revenue growth, expansion"
- "Identify technology adoption signals: new tools in their stack, integration announcements"

---

## Output

### For Small Inline Requests (1-5 companies)

Present results conversationally in a structured format:

```
🏢 Stripe (stripe.com)
  📈 Hiring: 47 open engineering roles, expanding platform team
  💰 Funding: Last raised Series I at $65B valuation
  🚀 Product: Launched Stripe Billing v3 in Q1 2026
  🤝 Partnership: New integration with Salesforce Revenue Cloud

🏢 Notion (notion.so)
  📈 Hiring: 12 open roles, heavy on enterprise sales
  👤 Leadership: New CRO hired from Figma in Jan 2026
```

### For Batch Requests (CSV output)

Build a CSV with original columns plus enrichment:

| Column              | Source                          |
|---------------------|---------------------------------|
| company_domain      | Input                           |
| company_name        | Input or enriched               |
| signals_summary     | Signaliz — condensed text       |
| hiring_signals      | Signaliz — hiring-specific      |
| funding_signals     | Signaliz — funding-specific     |
| news_signals        | Signaliz — recent news          |
| tech_stack          | Signaliz — detected tech        |
| raw_signals_json    | Full Signaliz response (JSON)   |
| (original columns)  | Preserved from input            |

**Structuring signal columns:** The raw Signaliz response contains rich
nested data. For CSV output, flatten the most useful fields into readable
text columns. Keep the full JSON in `raw_signals_json` for users who want
to parse it downstream.

Save to `/mnt/user-data/outputs/enriched_companies_[timestamp].csv` and
present with `present_files`.

### Summary Stats

```
🔍 Company Signal Enrichment Results
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Input:                  70 companies
Successfully enriched:  65 (92.9%)
No signals found:        3 (4.3%)
Errors:                  2 (2.9%)

Signal Distribution:
  📈 Hiring signals:     52 companies
  💰 Funding signals:    23 companies
  🚀 Product launches:   18 companies
  👤 Leadership changes: 11 companies
  🤝 Partnerships:        9 companies

Output: enriched_companies_20260309.csv
```

### Post-Output Guidance

- "Companies with active hiring signals are likely in growth mode — great timing for outreach."
- "Funding signals indicate budget availability. Recent Series B+ are high-value targets."
- "Leadership changes create buying windows — new executives often bring new tools."
- "You can use these signals to write personalized openers in your outbound sequences."
- If the user wants to find contacts at these companies next, suggest the signaliz-find-verified-emails skill.

---

## Error Handling

| Problem                                  | Fix                                                              |
|------------------------------------------|------------------------------------------------------------------|
| All Signaliz tools fail simultaneously   | Disconnect and reconnect the Signaliz MCP (Supabase cold start)  |
| `execute_primitive` times out            | Reduce batch to 10-15 records and retry                           |
| `enrich_company_signals` job stuck       | Wait 60s and re-poll. If stuck >3min, resubmit                    |
| `check_job_status` returns partial data  | Paginate through all pages                                        |
| No domain column found in CSV            | Ask the user which column contains company domains/websites       |
| Some companies return no signals         | Normal — not all companies have detectable recent signals         |
| Deep search taking very long             | Expected for large batches. Update user on progress               |

---

## Example Sessions

**Example 1 — CSV Upload:**
User: "Enrich these companies with signals" [uploads target_accounts.csv]
1. Inspect CSV → 120 rows, domain column = "Website"
2. Normalize, dedupe → 115 unique domains
3. Batch 1-25: execute_primitive
4. Batch 26-50: execute_primitive
5. Remaining 65: Signaliz:enrich_company_signals (async)
6. Poll check_job_status until complete
7. Compile results → output CSV with summary

**Example 2 — Inline Company List:**
User: "What signals do you see for Stripe, Notion, Linear, and Vercel?"
1. Call execute_primitive with 4 companies, enable_deep_search=true
2. Present results conversationally with signal breakdown per company

**Example 3 — Targeted Signal Hunt:**
User: "Find hiring signals for these 30 SaaS companies — I want to know who's growing their sales team" [uploads list.csv]
1. Inspect CSV → 30 companies
2. Batch via execute_primitive (2 batches of 15) with config:
   signal_types=["hiring"], research_prompt="Focus on sales and GTM hiring"
3. Filter results to only hiring signals
4. Output CSV sorted by number of open sales roles

**Example 4 — Research + Enrich:**
User: "Find the top 15 AI infrastructure companies and get me their signals"
1. Web search for AI infrastructure companies
2. Build domain list, confirm with user
3. Enrich via execute_primitive with enable_deep_search=true
4. Present results + CSV output
