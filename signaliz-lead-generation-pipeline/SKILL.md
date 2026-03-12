---
name: signaliz-lead-generation-pipeline
description: >
  End-to-end lead generation pipeline: find companies and people with Octave,
  find and verify emails with Signaliz, and push leads to Instantly campaigns.
  Use when someone asks to: run a lead gen pipeline, build and launch an
  outbound campaign, find leads and add them to a campaign, or do end-to-end
  outbound. Trigger on phrases like "find leads and email them", "build a
  campaign for [ICP]", "outbound pipeline", "prospect into [industry]",
  "full outbound flow", "lead gen pipeline", "build me a list and launch
  it", "create an outreach campaign targeting [persona]", or any request
  combining lead finding, email verification, and campaign launch. Always
  use this skill for full-pipeline requests. For individual steps, use
  signaliz-find-companies-octave, signaliz-find-people-octave,
  signaliz-find-verified-emails, or signaliz-company-signals.
metadata:
  author: signaliz
  version: '1.0'
license: MIT
---

# Lead Generation Pipeline

End-to-end outbound pipeline: **Find companies (Octave)** → **Find people
(Octave)** → **Find & verify emails (Signaliz)** → **Push to campaign
(Instantly)**.

---

## Before You Start

**Required:** The Signaliz MCP must be connected. This pipeline uses tools
from three integrated platforms — all accessed through Signaliz:

1. **Octave** — for finding companies and people
2. **Signaliz** — for finding and verifying emails, and company signals
3. **Instantly** — for campaign management and lead delivery

If tools aren't responding, tell the user to disconnect and reconnect the
Signaliz MCP in their settings (Supabase edge function cold starts can cause
all tools to fail at once — a reconnect fixes it).

**Tool loading:** Before making any MCP calls, use `tool_search` to load tool
definitions for each service. Do not guess parameter names.

---

## Pipeline Steps

### Step 1: Define Target Criteria

Ask the user for their Ideal Customer Profile (ICP) if not already provided:

**Company criteria:**
- Industry / vertical
- Employee count / company size
- Location / geography
- Funding stage
- Tech stack requirements
- Revenue range

**Contact criteria:**
- Target titles / roles (e.g., VP of Sales, Head of Engineering)
- Seniority level
- Department

**Volume:**
- How many leads they want
- Max per company (default: 1-2 contacts per company)

**Campaign details (optional — can be defined at Step 5):**
- Campaign name
- Subject lines and email body
- Number of follow-up steps
- Tone (conversational, professional, direct)

Example:
```
User: "Build an outbound campaign targeting VP of Sales at Series B+
SaaS companies with 50-500 employees"
→ ICP: SaaS, 50-500 employees, Series B+
→ Persona: VP of Sales
→ Volume: default 20-50 leads
```

Confirm the criteria with the user before proceeding.

---

### Step 2: Find Companies (Octave)

Use Octave to build the target account list:

1. **`find_company`** — search by industry, size, location criteria
2. **`find_similar_companies`** — expand from seed accounts if provided
3. **`enrich_company`** — get firmographic details for each match
4. **`qualify_company`** — score against ICP (optional but recommended)

**Batch limits:** Cap Octave enrichment at 50 companies per session.

**Progress report:**
```
Step 2 ✅ — Found 47 companies matching ICP criteria
  43 strong fit, 4 moderate fit
  Top matches: Stripe, Notion, Linear, Figma, Vercel...
```

---

### Step 3: Find People (Octave)

Use Octave to find contacts at the target companies:

1. **`find_person`** — search by title/role at each company
2. **`enrich_person`** — get detailed contact profiles (LinkedIn, seniority)
3. **`qualify_person`** — score against buyer persona (optional)

**Batch limits:** Cap at 200 person lookups per session. For larger lists,
suggest splitting into multiple runs.

**Progress report:**
```
Step 3 ✅ — Found 52 contacts across 47 companies
  VP of Sales: 38
  Head of Sales: 9
  Director of Sales: 5
  No match at 3 companies
```

---

### Step 4: Find & Verify Emails (Signaliz)

Use Signaliz to find and verify professional emails for each contact:

**For 1-25 contacts** — use `execute_primitive`:
```json
{
  "capability_id": "find_verified_emails",
  "input_data": [
    {
      "first_name": "Jane",
      "last_name": "Smith",
      "company_domain": "acme.com"
    }
  ]
}
```

**For 26-5,000 contacts** — use `find_and_verify_emails`:
- Submit all contacts in a single async job
- Returns a `job_id` immediately
- Poll with `check_job_status` (respect `next_poll_after_seconds`)
- Use `page_size: 500` when retrieving results

**Filter results:**
- **Keep:** contacts with `email_verified: true` (valid)
- **Flag:** contacts with `email_is_catch_all: true` (include but note risk)
- **Drop:** contacts with no email found or `invalid` status

**Progress report:**
```
Step 4 ✅ — Email results for 52 contacts
  ✅ Valid:      41 (78.8%)
  ⚠️ Catch-all:   5 (9.6%)
  ❌ Not found:    6 (11.5%)
  46 contacts ready for campaign
```

---

### Step 5: Push to Instantly Campaign

#### 5a. Select or Create Campaign

**List existing campaigns:**
Use `list_campaigns` to show available campaigns. Let the user choose one.

**Or create a new campaign:**
Use `create_campaign` with:
- `name` — campaign name
- Subject line and email body (if provided in Step 1)
- Sequence steps and timing

#### 5b. Add Leads to Campaign

Use `add_leads_to_campaign_or_list_bulk` to push verified leads:

```json
{
  "campaign_id": "campaign-uuid",
  "leads": [
    {
      "email": "jane@acme.com",
      "first_name": "Jane",
      "last_name": "Smith",
      "company_name": "Acme Corp",
      "personalization": "Noticed Acme recently raised a Series B..."
    }
  ]
}
```

**Batching:** Instantly accepts 1,000 leads per bulk call. For larger lists,
batch in chunks of 1,000 with 2-second gaps between batches.

Include enrichment data as custom variables for personalization:
- Company signals (hiring, funding, product launches)
- Firmographic details (industry, size, location)
- Contact details (title, seniority, department)

#### 5c. Activate Campaign

**Always ask the user before activating.** Never activate a campaign without
explicit user confirmation.

```
Ready to activate campaign "Q1 Outbound — VP Sales at SaaS"
with 46 verified leads. Shall I activate it?
```

Use `activate_campaign` only after the user confirms.

---

## Progress Reporting

After each step, show cumulative progress:

```
Pipeline Progress:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Step 1: ✅ ICP defined — VP Sales at Series B+ SaaS (50-500 employees)
Step 2: ✅ 47 companies found via Octave
Step 3: ✅ 52 contacts found at target companies
Step 4: ✅ 41 verified emails (5 catch-all, 6 not found)
Step 5: ⏳ Ready to push 46 leads to Instantly...
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## Final Output

Present a comprehensive summary:

```
🚀 Lead Generation Pipeline — Complete
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
ICP:               VP of Sales at Series B+ SaaS (50-500 employees)
Companies found:   47 accounts matching ICP
Contacts found:    52 decision makers identified
Emails verified:   41 valid + 5 catch-all = 46 campaign-ready
Leads pushed:      46 leads added to "Q1 Outbound — VP Sales at SaaS"
Campaign status:   Active ✅ (or Draft — pending activation)

Output files:
  📄 pipeline_leads_20260311.csv — Full enriched lead list
  📄 pipeline_summary_20260311.csv — Campaign-ready contacts only
```

**Post-pipeline guidance:**
- "Monitor campaign performance in Instantly — track open rates, reply rates, and bounces."
- "Catch-all emails are flagged — watch for bounces in the first 48 hours and pull any that bounce."
- "Want to run this pipeline again for a different ICP or persona?"
- "Want to add company signals as personalization variables for each lead?"

---

## Error Handling

| Step | Problem                             | Fix                                                    |
|------|-------------------------------------|--------------------------------------------------------|
| 2    | Octave company search returns 0     | Broaden criteria (larger size range, more industries)  |
| 3    | Octave person search returns 0      | Try broader titles (e.g., "Sales" instead of "VP Sales")|
| 4    | Signaliz tools fail                 | Reconnect Signaliz MCP (cold start — wait 5s, retry)  |
| 4    | Low email hit rate (<50%)           | Add LinkedIn URLs from Octave enrichment for accuracy  |
| 5    | Instantly tools fail                | Check Instantly MCP connection and API key             |
| 5    | Campaign not found                  | Use `list_campaigns` and let user select               |
| All  | Rate limit errors (429)             | Wait 30 seconds and retry; reduce batch size           |

### Recovery Protocol

For any batch failure:
1. Attempt up to 3 retries with exponential backoff (5s → 10s → 20s)
2. Rate limit errors: wait 30 seconds before retry
3. Partial failures: collect successes, retry only failed records
4. After 3 retries: report failures, offer manual retry
5. Never silently drop records — always report what failed and why

---

## Example Sessions

**Example 1 — Full Pipeline from ICP:**
User: "Build an outbound campaign targeting VP of Engineering at fintech companies with 100-500 employees"
1. Confirm ICP → fintech, 100-500 employees, VP of Engineering
2. Octave find_company → 35 fintech companies found
3. Octave find_person → 31 VPs of Engineering found
4. Signaliz find_verified_emails → 25 valid, 3 catch-all, 3 not found
5. Create Instantly campaign → 28 leads loaded
6. User confirms → Campaign activated

**Example 2 — Pipeline from CSV:**
User: "Here's my target account list — find VP Sales at each, verify emails, and push to my Instantly campaign" [uploads accounts.csv]
1. Parse CSV → 50 companies
2. Skip Octave company search (accounts already provided)
3. Octave find_person → VP of Sales at each company → 43 contacts
4. Signaliz verify → 36 valid, 4 catch-all
5. Push 40 leads to user's selected Instantly campaign

**Example 3 — Pipeline with Seed Companies:**
User: "Find 30 companies like Stripe and Ramp, then find their Head of Sales, verify emails, and set up a campaign"
1. Octave find_similar_companies for each seed → 30 companies
2. Octave find_person → Head of Sales at each → 26 contacts
3. Signaliz verify → 21 valid, 3 catch-all
4. Create Instantly campaign with 24 leads
5. Await user activation
