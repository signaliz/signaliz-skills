---
name: signaliz-account-intel-report
description: >
  Generate a comprehensive account intelligence dossier for a target company
  using Signaliz and Octave. Use when someone asks to: research a target
  account in depth, build a company dossier, prepare an account brief, do a
  deep-dive, or prep for a sales call. Trigger on phrases like "tell me
  everything about [company]", "account research for [company]", "build a
  dossier on [company]", "company intel report", "research [company] before
  my call", "account brief", "what do we know about [company]", or any
  request for thorough single-company research beyond basic signals. Combines
  Signaliz company signals, Octave firmographic and technographic enrichment,
  decision-maker mapping, and AI synthesis into a structured deliverable. For
  batch company research (many companies, light detail), use
  signaliz-company-signals instead.
metadata:
  author: signaliz
  version: '1.0'
license: MIT
---

# Account Intelligence Report via Signaliz

Generate a deep, structured intelligence dossier for a single target company —
combining Signaliz signals, Octave firmographic/technographic enrichment,
decision-maker mapping, and AI-powered synthesis into an actionable account
brief.

---

## Before You Start

**Required:** The Signaliz MCP must be connected. This skill uses both
Signaliz and Octave capabilities. If tools like `execute_primitive`,
`enrich_company`, or `find_person` aren't responding, tell the user to
disconnect and reconnect the Signaliz MCP in their settings.

**Tool loading:** Before making any MCP calls, use `tool_search` to load the
correct tool definitions for both Signaliz and Octave tools. Do not guess
parameter names.

**Scope:** This skill is designed for deep research on 1-5 companies. For
batch enrichment of 10+ companies, redirect to signaliz-company-signals.

---

## Input

The user provides a target company by any of:
- Company name (e.g., "Stripe")
- Company domain (e.g., "stripe.com")
- Company URL (e.g., "https://stripe.com")
- A brief description (e.g., "the payments company that just raised Series I")

If ambiguous, clarify which company the user means before proceeding.

**Optional context the user may provide:**
- Why they're researching (sales call prep, competitive analysis, partnership eval)
- Specific areas of interest (tech stack, hiring, funding, leadership)
- Their product/service (helps tailor the "recommended approach" section)
- Target persona they want to reach (helps prioritize decision-maker mapping)

---

## Execution Steps

### Step 1: Company Enrichment (Octave)

Call `enrich_company` with the company domain:

```json
{
  "domain": "stripe.com"
}
```

Extract:
- Company name, description, and tagline
- Industry and sub-industry
- Employee count and growth trajectory
- Headquarters location and office footprint
- Founded date
- Revenue estimate / range
- Funding history (rounds, investors, valuation)
- Tech stack (technologies in use)
- Key products and services

### Step 2: Signal Enrichment (Signaliz)

Call `execute_primitive` with `enrich_company_signals` and deep search enabled:

```json
{
  "capability_id": "enrich_company_signals",
  "input_data": [
    {"company_domain": "stripe.com"}
  ],
  "config": {
    "enable_deep_search": true
  }
}
```

Extract and categorize signals:
- **Hiring signals:** Open roles, team growth, new departments
- **Funding signals:** Recent rounds, investors, valuation changes
- **Product signals:** Launches, features, major releases
- **Partnership signals:** Integrations, strategic alliances
- **Leadership signals:** New executives, departures, reorgs
- **Expansion signals:** New offices, markets, geographies
- **Acquisition signals:** M&A activity (acquiring or being acquired)
- **News signals:** Press coverage, awards, regulatory actions

### Step 3: Decision-Maker Mapping (Octave)

Identify key contacts at the company. Search for the most common buying
committee roles:

1. **Economic buyer** — CFO, VP of Finance, Head of Procurement
2. **Technical buyer** — CTO, VP of Engineering, Head of IT
3. **Champion / User buyer** — varies by product (e.g., VP of Sales, VP of Marketing)
4. **Executive sponsor** — CEO, COO, President

If the user specified a target persona, prioritize that role. Otherwise, map
the top 3-5 decision makers.

For each contact found via `find_person`, extract:
- Full name and title
- LinkedIn URL
- Seniority and department
- Tenure at company (if available)

### Step 4: Competitive Landscape

Use web search to identify 3-5 direct competitors. For each competitor, note:
- Company name and domain
- How they compete (product overlap, market overlap)
- Key differentiators

If the user mentioned their own product, frame the competitive landscape from
the perspective of the target company's buying decision.

### Step 5: AI Synthesis — Recommended Approach

Based on all collected intelligence, synthesize:

1. **Timing assessment:** Is now a good time to engage? (Based on signals —
   recent funding = budget available, hiring = growing, leadership change =
   new vendor evaluation window)

2. **Entry points:** Which signals or events create a natural conversation
   opener? (e.g., "Congratulations on the Series C — as you scale your
   engineering team, we help with...")

3. **Recommended contacts:** Who to reach first and why (based on title,
   seniority, and relevance to your offering)

4. **Potential objections:** What might block a deal? (e.g., they just hired
   a Head of [your category], they have a competitor tool in their tech stack)

5. **Talk track hooks:** 2-3 specific talking points tied to real signals

---

## Output Format

Present the dossier in a structured, readable format:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📊 Account Intelligence Report: Stripe
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

🏢 COMPANY OVERVIEW
  Name:           Stripe, Inc.
  Domain:         stripe.com
  Industry:       Financial Services / Payments Infrastructure
  Founded:        2010
  HQ:             San Francisco, CA
  Employees:      8,000+ (↑ 15% YoY)
  Revenue:        ~$14B (estimated)
  Funding:        Series I — $6.5B raised, $65B valuation
  Description:    Online payment processing platform for internet businesses

🔧 TECH STACK
  Languages:      Ruby, Go, Python, Java
  Infrastructure: AWS, Kubernetes, Terraform
  Tools:          Salesforce, Slack, Jira, Datadog

📈 ACTIVE SIGNALS
  🔥 Hiring:       47 open engineering roles, 12 sales roles
  💰 Funding:      Series I closed Q3 2024 — $65B valuation
  🚀 Product:      Launched Stripe Billing v3 in Q1 2026
  🤝 Partnership:  New integration with Salesforce Revenue Cloud
  👤 Leadership:   New CRO joined from Datadog in Jan 2026

👥 KEY DECISION MAKERS
  1. [Name] — CTO
     LinkedIn: [url]
  2. [Name] — VP of Engineering
     LinkedIn: [url]
  3. [Name] — VP of Sales
     LinkedIn: [url]

⚔️ COMPETITIVE LANDSCAPE
  • Adyen — European competitor, strong in enterprise
  • Square (Block) — SMB focus, expanding upmarket
  • Braintree (PayPal) — Legacy competitor, losing market share

🎯 RECOMMENDED APPROACH
  Timing:      Strong — recent funding + active hiring = growth mode
  Entry point: Series I momentum + engineering team expansion
  First contact: [Name], VP of Engineering (most relevant to your offering)
  Talk track:
    1. "As you scale post-Series I, [your product] helps teams like yours..."
    2. "Noticed you're hiring 47 engineers — when teams grow that fast..."
    3. "Your new Salesforce integration suggests you're investing in revenue ops..."

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Generated: [timestamp] | Sources: Signaliz, Octave, web
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### CSV / File Output

Also offer to save the report as a structured file:
- **Full report** — Markdown or PDF format for sharing with the sales team
- **Contacts CSV** — Decision makers with LinkedIn URLs, ready for email finding
- **Signals CSV** — Structured signal data for CRM import

---

## Post-Output Guidance

After presenting the report, suggest next steps:
- "Want me to find verified emails for these decision makers?" → signaliz-find-verified-emails
- "Want to build an outreach sequence based on these signals?" → signaliz-lead-generation-pipeline
- "Want me to research more companies like this one?" → signaliz-find-companies-octave
- "Want a similar report for [competitor]?" → Run this skill again

---

## Multi-Company Mode

If the user provides 2-5 companies, generate a report for each. For 3+
companies, also include a comparison matrix:

| Dimension        | Company A | Company B | Company C |
|------------------|-----------|-----------|-----------|
| Size             | 8,000     | 1,200     | 450       |
| Funding stage    | Series I  | Series D  | Series B  |
| Hiring velocity  | High      | Medium    | High      |
| Tech overlap     | Yes       | No        | Partial   |
| Recommended first| ✅        |           |           |

---

## Error Handling

| Problem                                | Fix                                                              |
|----------------------------------------|------------------------------------------------------------------|
| All MCP tools fail simultaneously      | Disconnect and reconnect the Signaliz MCP (Supabase cold start)  |
| Octave returns minimal data            | Supplement with web search for company info                      |
| No signals found by Signaliz           | Normal for smaller/private companies — note in report            |
| Decision makers not found              | Try broader title searches; supplement with web/LinkedIn search  |
| Company not found by name              | Ask for the domain; try alternate names or subsidiaries          |

---

## Example Sessions

**Example 1 — Sales Call Prep:**
User: "I have a call with the VP of Sales at Notion tomorrow — research them for me"
1. Enrich Notion via Octave (firmographics, tech stack)
2. Enrich via Signaliz (signals: hiring, funding, product)
3. Find VP of Sales + other decision makers via Octave
4. Web search for competitive landscape
5. Synthesize recommended approach with talk track hooks
6. Present full dossier

**Example 2 — Account Prioritization:**
User: "Build intel reports for Stripe, Notion, and Linear — I need to decide which to pursue first"
1. Generate full dossier for each company
2. Include comparison matrix
3. Recommend priority order based on signal strength and timing
4. Offer to start the outbound pipeline for the top pick

**Example 3 — Partnership Evaluation:**
User: "Research Figma for a potential partnership — what should I know?"
1. Full enrichment (Octave + Signaliz)
2. Focus on partnership signals and strategic direction
3. Map business development and partnerships contacts
4. Synthesize partnership fit assessment instead of sales approach
