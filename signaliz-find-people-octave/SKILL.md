---
name: signaliz-find-people-octave
description: >
  Find people and contacts at companies using Octave via Signaliz. Use this
  skill whenever someone asks to: find people, search for contacts, find
  decision makers, look up people at a company, find leads, find prospects,
  build a contact list, or find people by title or role. Trigger on phrases
  like "who works at [company]", "find the VP of Sales at", "find people
  like", "decision makers at [company]", "build a prospect list", "find
  contacts at these companies", "who is the Head of Marketing at [company]",
  "find leads at [company]", or any request to discover contacts by name,
  title, role, or company. Also trigger when someone uploads a CSV of
  companies and wants to find contacts at them. Always use this skill when
  the goal is finding people or contacts — even if the user doesn't mention
  Octave or Signaliz by name. For finding companies first, use
  signaliz-find-companies-octave. For finding emails after identifying
  contacts, use signaliz-find-verified-emails.
metadata:
  author: signaliz
  version: '1.0'
license: MIT
---

# Find People with Octave via Signaliz

Search for people by name, company, title, role, seniority, or similarity to
existing contacts using Octave's contact intelligence — accessed through the
Signaliz MCP.

---

## Before You Start

**Required:** The Signaliz MCP must be connected. Octave people intelligence
is accessed through Signaliz's MCP tools. If tools like `find_person` or
`enrich_person` aren't responding, tell the user to disconnect and reconnect
the Signaliz MCP in their settings (Supabase edge function cold starts can
cause all tools to fail at once — a reconnect fixes it).

**Tool loading:** Before making any MCP calls, use `tool_search` to load the
correct tool definitions. Look for tools prefixed with `mcp__Octave__`,
`mcp__Signaliz__`, or the generic `execute_primitive`. Do not guess parameter
names.

---

## Input Modes

### Mode A: Role-Based Search

The user wants to find people at specific companies by title, role, or
department. Parse the request:

| Criterion       | Examples                                              |
|-----------------|-------------------------------------------------------|
| Title/role      | VP of Sales, Head of Engineering, CMO, SDR Manager    |
| Seniority       | C-level, VP, Director, Manager, Individual Contributor|
| Department      | Sales, Engineering, Marketing, Operations, Finance    |
| Company         | By name or domain                                     |

Example:
```
User: "Find the VP of Sales at Stripe, Notion, and Linear"
→ title: VP of Sales, companies: [stripe.com, notion.so, linear.app]
```

### Mode B: Named Person Lookup

The user wants to find a specific person:

```
User: "Find Sarah Chen at Notion"
→ first_name: Sarah, last_name: Chen, company: Notion
```

### Mode C: Similar People Search

The user wants to find contacts similar to an existing one:

```
User: "Find people similar to our champion at Stripe — she's a Senior Director of Platform Engineering"
→ Use find_similar_people with the seed contact profile
```

### Mode D: CSV Upload

The user uploads a CSV with company names/domains and wants contacts found at
each. Inspect and map columns:

| Useful Field     | Common CSV Variants                                    |
|------------------|--------------------------------------------------------|
| `company_domain` | Domain, Website, website, company_url                  |
| `company_name`   | Company, Company Name, Organization, Account           |
| `target_title`   | Title, Role, Target Role, Persona                      |

Parse the file, extract companies, and find contacts at each.

### Mode E: Inline Company List

The user provides companies directly in conversation with a target role:

```
User: "Find the Head of Marketing at these 10 companies: [list]"
```

Collect the companies and target title, then execute.

---

## Execution: Finding People

### Choosing the Right Tool

| Goal                              | Tool                   | Notes                           |
|-----------------------------------|------------------------|---------------------------------|
| Find a specific person            | `find_person`          | Name + company                  |
| Find people at a company by role  | `find_person`          | Title/role + company            |
| Find similar contacts             | `find_similar_people`  | Seed contact profile            |
| Deep contact enrichment           | `enrich_person`        | Full profile, employment history|
| Buyer persona qualification       | `qualify_person`       | Score against persona criteria  |

### Finding People at Companies

For each target company, call `find_person` with the role/title criteria:

```json
{
  "company_name": "Stripe",
  "domain": "stripe.com",
  "title": "VP of Sales"
}
```

If the user wants multiple roles at each company, make separate calls per
role and consolidate.

### Deep Enrichment

After finding contacts, use `enrich_person` for additional detail:

```json
{
  "linkedin_url": "https://linkedin.com/in/janesmith"
}
```

Returns: employment history, education, skills, social profiles, and more.

### Buyer Persona Qualification

Use `qualify_person` to score contacts against the user's buyer persona:

```json
{
  "name": "Jane Smith",
  "title": "VP of Sales",
  "company": "Stripe",
  "persona_criteria": "B2B SaaS sales leader, manages SDR/AE teams, owns revenue targets"
}
```

### Batch Processing

When processing lists of companies:
- Call `find_person` for each company sequentially
- Add a 1-second gap between calls to respect rate limits
- Cap at 200 person lookups per session
- For lists exceeding this, suggest breaking into batches across sessions

---

## Output

### For Small Requests (1-5 contacts)

Present results conversationally:

```
👤 Jane Smith — VP of Sales
  Company: Stripe (stripe.com)
  Location: San Francisco, CA
  Seniority: VP / Executive
  Department: Sales
  LinkedIn: linkedin.com/in/janesmith

👤 Marcus Johnson — Head of Sales
  Company: Notion (notion.so)
  Location: New York, NY
  Seniority: Director / Head
  Department: Sales
  LinkedIn: linkedin.com/in/marcusjohnson
```

### For Batch Requests (CSV output)

Build a CSV with these columns:

| Column           | Source                                  |
|------------------|-----------------------------------------|
| first_name       | Octave result                           |
| last_name        | Octave result                           |
| title            | Octave result                           |
| company_name     | Input or Octave result                  |
| company_domain   | Input or Octave result                  |
| location         | Octave result                           |
| seniority        | Octave result                           |
| department       | Octave result                           |
| linkedin_url     | Octave result                           |
| persona_score    | Octave qualification (if requested)     |
| (original cols)  | Preserved from input                    |

Save and present the file to the user.

### Summary Stats

```
👤 People Search Results
━━━━━━━━━━━━━━━━━━━━━━━━
Target role:         VP of Sales
Companies searched:  25
Contacts found:      22 (88%)
Not found:            3 companies (no matching role)

Output: target_contacts_20260311.csv
```

### Post-Output Guidance

After presenting results, suggest natural next steps:
- "Want me to find verified emails for these contacts?" → signaliz-find-verified-emails
- "Want me to check company signals for their employers?" → signaliz-company-signals
- "Want to run the full pipeline — verify emails and push to a campaign?" → signaliz-lead-generation-pipeline
- "Want me to score these contacts against your buyer persona?" → Use qualify_person

---

## Error Handling

| Problem                                | Fix                                                              |
|----------------------------------------|------------------------------------------------------------------|
| All MCP tools fail simultaneously      | Disconnect and reconnect the Signaliz MCP (Supabase cold start)  |
| Octave tools not found                 | Ensure Octave MCP is connected; check API key                    |
| `find_person` returns no results       | Try broader title search or check company name/domain spelling   |
| Multiple people returned for one role  | Present all matches, let user select the best fit                |
| Rate limiting                          | Add 2-second gaps between calls; reduce batch size               |
| LinkedIn URL missing from results      | Normal — not all contacts have indexed LinkedIn profiles         |

---

## Example Sessions

**Example 1 — Role-Based Search:**
User: "Find the VP of Sales at Stripe, Notion, Linear, Figma, and Vercel"
1. Call `find_person` with title "VP of Sales" at each company
2. Enrich results with `enrich_person` for LinkedIn URLs
3. Present 5 contact cards with full details
4. Offer to find verified emails next

**Example 2 — Named Person Lookup:**
User: "Find Sarah Chen at Notion"
1. Call `find_person` with first_name="Sarah", last_name="Chen", company="Notion"
2. Return full contact profile
3. Offer to find her email or find similar contacts

**Example 3 — CSV Company List:**
User: "Find the Head of Marketing at each of these companies" [uploads accounts.csv]
1. Inspect CSV → 30 companies with "Domain" column
2. Call `find_person` with title "Head of Marketing" at each domain
3. Found 26 contacts, 4 companies with no match
4. Output CSV with contact details + offer email finding

**Example 4 — Similar People:**
User: "Find 10 people similar to Jane Smith, VP of Platform Engineering at Stripe"
1. Call `find_similar_people` with Jane's profile as seed
2. Present 10 similar contacts ranked by similarity
3. Offer enrichment or email finding for selected contacts
