---
name: signaliz-find-companies-octave
description: >
  Find and research companies using Octave via Signaliz. Use this skill
  whenever someone asks to: find companies, search for companies, discover
  target accounts, build an account list, find companies like [company], find
  similar companies, research companies by criteria, or match against an ICP.
  Trigger on phrases like "find me companies in [industry]", "who are the
  competitors of", "companies like [company]", "build a target account list",
  "ICP matching", "find SaaS companies", "search for companies that use
  [technology]", "find accounts in [location]", or any request to discover
  or research companies by firmographic criteria. Also trigger when someone
  uploads a CSV of company names or domains and wants enrichment or similar
  company expansion. Always use this skill when the goal is finding or
  researching companies — even if the user doesn't mention Octave or Signaliz
  by name. For finding people at companies, use signaliz-find-people-octave.
  For finding emails, use signaliz-find-verified-emails.
metadata:
  author: signaliz
  version: '1.0'
license: MIT
---

# Find Companies with Octave via Signaliz

Search for companies by name, domain, industry, size, tech stack, or similarity
to existing accounts using Octave's company intelligence — accessed through the
Signaliz MCP.

---

## Before You Start

**Required:** The Signaliz MCP must be connected. Octave company intelligence
is accessed through Signaliz's MCP tools. If tools like `execute_primitive` or
`find_company` aren't responding, tell the user to disconnect and reconnect the
Signaliz MCP in their settings (Supabase edge function cold starts can cause
all tools to fail at once — a reconnect fixes it).

**Tool loading:** Before making any MCP calls, use `tool_search` to load the
correct tool definitions. Look for tools prefixed with `mcp__Octave__`,
`mcp__Signaliz__`, or the generic `execute_primitive`. Do not guess parameter
names.

---

## Input Modes

### Mode A: Criteria-Based Search

The user describes target companies using natural language. Parse their request
into structured search criteria:

| Criterion       | Examples                                              |
|-----------------|-------------------------------------------------------|
| Industry        | SaaS, fintech, healthcare, e-commerce                 |
| Size            | 50-500 employees, mid-market, enterprise              |
| Location        | US, Bay Area, Europe, APAC                            |
| Funding stage   | Series A+, Series B+, bootstrapped                    |
| Tech stack      | Uses Salesforce, runs on AWS, Snowflake customer      |
| Revenue range   | $10M-$50M ARR, pre-revenue, $100M+                   |

Example:
```
User: "Find B2B SaaS companies with 50-500 employees that use Salesforce"
→ industry: B2B SaaS, size: 50-500, tech: Salesforce
```

### Mode B: Seed Company Expansion

The user provides one or more seed companies and wants to find similar ones:

```
User: "Find companies similar to Stripe, Plaid, and Ramp"
→ Use find_similar_companies with each seed domain
```

### Mode C: CSV Upload

The user uploads a CSV with company names or domains. Inspect the file and
identify the relevant columns:

| Required Field   | Common CSV Variants                                    |
|------------------|--------------------------------------------------------|
| `company_domain` | Domain, Website, website, company_url, domain, URL     |
| `company_name`   | Company, Company Name, Name, Organization, Account     |

**Pre-processing:**
1. Normalize domains: strip `https://`, `http://`, `www.`, trailing slashes
2. Lowercase all domains
3. Deduplicate by domain
4. Remove empty rows

Report what you found:
```
Found 35 companies with domain column "Website"
Removed 2 duplicates, 1 empty row
32 companies ready for enrichment
```

### Mode D: Inline List

The user provides company names or domains directly in conversation. Collect
them and build the input list.

---

## Execution: Finding Companies

### Choosing the Right Tool

| Goal                          | Tool                     | Notes                           |
|-------------------------------|--------------------------|---------------------------------|
| Search by name or domain      | `find_company`           | Returns company details         |
| Find similar companies        | `find_similar_companies` | Takes a seed company            |
| Deep enrichment               | `enrich_company`         | Firmographics, tech, funding    |
| ICP qualification scoring     | `qualify_company`        | Scores against criteria         |

### Single Company Lookup

```json
{
  "company_name": "Stripe",
  "domain": "stripe.com"
}
```

Returns: industry, employee count, location, tech stack, funding, description.

### Similar Company Search

```json
{
  "domain": "stripe.com"
}
```

Returns a list of companies with similar characteristics — industry, size,
tech stack, and business model.

### Batch Enrichment (CSV or List)

For lists of companies, call `enrich_company` for each domain. Process
sequentially with a 1-second gap to respect rate limits. Cap at 50 companies
per session for Octave enrichment.

For larger lists, use Signaliz's `execute_primitive` with
`capability_id: "enrich_company_signals"` which handles batches of up to 25:

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

### ICP Qualification

If the user has defined ICP criteria, use `qualify_company` to score each
company against those criteria. Present results ranked by fit score.

---

## Output

### For Small Requests (1-5 companies)

Present results conversationally:

```
🏢 Stripe (stripe.com)
  Industry: Financial Services / Payments
  Size: 8,000+ employees
  HQ: San Francisco, CA
  Funding: Series I — $65B valuation
  Tech: Ruby, Go, React, AWS
  Description: Online payment processing platform for internet businesses

🏢 Plaid (plaid.com)
  Industry: Financial Services / Banking Infrastructure
  Size: 1,200+ employees
  HQ: San Francisco, CA
  Funding: Series D — $13.4B valuation
  Tech: Python, Kotlin, AWS, Kubernetes
  Description: Financial data network connecting apps to bank accounts
```

### For Batch Requests (CSV output)

Build a CSV with these columns:

| Column           | Source                                  |
|------------------|-----------------------------------------|
| company_domain   | Input or enriched                       |
| company_name     | Octave enrichment                       |
| industry         | Octave enrichment                       |
| employee_count   | Octave enrichment                       |
| hq_location      | Octave enrichment                       |
| funding_stage    | Octave enrichment                       |
| tech_stack       | Octave enrichment (comma-separated)     |
| description      | Octave enrichment                       |
| icp_score        | Octave qualification (if requested)     |
| (original cols)  | Preserved from input                    |

Save and present the file to the user.

### Summary Stats

```
🏢 Company Search Results
━━━━━━━━━━━━━━━━━━━━━━━━━
Search criteria:     B2B SaaS, 50-500 employees, US-based
Companies found:     47
Enriched:            47 (100%)
ICP qualified:       32 strong fit, 10 moderate, 5 weak

Output: target_accounts_20260311.csv
```

### Post-Output Guidance

After presenting results, suggest natural next steps:
- "Want me to find decision makers at these companies?" → signaliz-find-people-octave
- "Want me to find verified emails for contacts here?" → signaliz-find-verified-emails
- "Want me to check what signals these companies are showing?" → signaliz-company-signals
- "Want to run the full pipeline — find people, verify emails, and launch a campaign?" → signaliz-lead-generation-pipeline

---

## Error Handling

| Problem                                | Fix                                                              |
|----------------------------------------|------------------------------------------------------------------|
| All MCP tools fail simultaneously      | Disconnect and reconnect the Signaliz MCP (Supabase cold start)  |
| Octave tools not found                 | Ensure Octave MCP is connected; check API key                    |
| `find_company` returns no results      | Try alternate spellings, use domain instead of name              |
| `find_similar_companies` thin results  | Try a different seed company or broaden criteria                 |
| Rate limiting on enrichment            | Reduce batch pace; add 2-second gaps between calls               |
| CSV missing domain column              | Ask user which column contains company names/websites            |

---

## Example Sessions

**Example 1 — Criteria Search:**
User: "Find fintech companies in the US with 100-1000 employees"
1. Parse criteria → industry: fintech, location: US, size: 100-1000
2. Call `find_company` with criteria
3. Enrich top results with `enrich_company`
4. Present results with firmographic breakdown

**Example 2 — Similar Company Expansion:**
User: "Find 20 companies similar to Notion and Linear"
1. Call `find_similar_companies` with notion.so as seed
2. Call `find_similar_companies` with linear.app as seed
3. Deduplicate, merge, rank by similarity score
4. Present top 20 with enrichment details

**Example 3 — CSV Enrichment:**
User: "Enrich these companies" [uploads target_list.csv]
1. Inspect CSV → 40 companies with "Website" column
2. Normalize domains, deduplicate → 38 unique
3. Batch enrich via execute_primitive (2 batches of 19)
4. Output enriched CSV with firmographic columns added

**Example 4 — ICP Qualification:**
User: "Which of these companies fit our ICP: Series B+ SaaS with 200+ employees?"
1. Parse ICP criteria
2. Enrich each company if not already enriched
3. Qualify each against ICP via `qualify_company`
4. Rank by fit score, present top matches first
