---
name: signaliz-find-verified-emails
description: >
  Find verified professional email addresses for contacts using Signaliz MCP.
  Use this skill whenever someone asks to: find emails for a list of people,
  look up someone's work email, discover email addresses for contacts, get
  verified emails for outreach, or find professional emails from names and
  company domains. Trigger on phrases like "find emails for", "get me emails
  for these contacts", "look up email for [name] at [company]", "find their
  work email", "email finder", "discover emails", "I need emails for these
  people", "find emails from this CSV", or any request to go from a person's
  name + company to a verified email address. Also trigger when someone uploads
  a CSV containing names and companies and asks to find or append emails.
  Always use this skill when the goal is finding email addresses — even if the
  user doesn't mention Signaliz by name. Do NOT use this skill for verifying
  emails that the user already has — use signaliz-verify-emails instead.
---

# Find Verified Emails via Signaliz

Turn a list of contacts (name + company) into verified email addresses using
Signaliz's email finding and verification engine. Accepts input from a CSV
upload or directly from Claude (inline data, web research, Clay results, etc.).

---

## Before You Start

**Required:** The Signaliz MCP must be connected. If tools like
`find_emails_with_verification` or `execute_primitive` aren't responding,
tell the user to disconnect and reconnect the Signaliz MCP in their Claude
settings (Supabase edge function cold starts can cause all tools to fail at
once — a reconnect fixes it).

**Tool loading:** Before making any Signaliz MCP calls, use `tool_search` to
load the correct tool definitions. Do not guess parameter names.

---

## Input Modes

This skill handles two input modes. Determine which one applies before
proceeding.

### Mode A: CSV Upload

The user uploads a CSV containing contact data. Inspect it and map columns:

| Required Field   | Common CSV Variants                                      |
|------------------|----------------------------------------------------------|
| `first_name`     | FirstName, First Name, first, fname, First               |
| `last_name`      | LastName, Last Name, last, lname, Last                   |
| `company_domain` | Domain, Website, website, company_url, Company Domain    |

Optional but helpful:
| Field            | Common Variants                                          |
|------------------|----------------------------------------------------------|
| `company_name`   | Company, company, Organization, Account, Company Name    |
| `linkedin_url`   | LinkedIn, linkedin_url, LinkedIn URL, Profile URL        |
| `full_name`      | Name, Full Name, FullName, Contact Name                  |

**If the CSV has `full_name` but not separate first/last:** Split it. First
token = first_name, everything after = last_name.

**If the CSV has emails already:** Ask the user whether they want to find
*additional/alternative* emails or just verify what they have (→ redirect to
signaliz-verify-emails skill).

**If the CSV is missing `company_domain` but has `company_name`:** Attempt to
derive the domain by searching `"[Company Name] official website"` for each
company name. If you can't find the domain confidently, flag the row and skip
it.

Report what you found:
```
Found 47 contacts with columns: first_name, last_name, company_domain, company_name
3 rows missing company_domain — will attempt to derive from company_name
2 rows missing last_name — will skip (required for email finding)
42 contacts ready for email finding
```

### Mode B: Inline / Conversational Input

The user provides contacts directly in conversation, or asks Claude to
research contacts first (e.g., "find emails for the VP Sales at these 10
companies"). In this case:

1. Collect or research the required fields: first_name, last_name,
   company_domain
2. If only company + title is given with no names, research contact names
   via web search: `"[Company Name] [Target Title] LinkedIn"`
3. Build the input list and confirm with the user before proceeding

---

## Execution: Finding Emails

### Choosing the Right Tool

Signaliz offers two paths for finding emails. Choose based on batch size:

| Batch Size | Tool                                | Pattern           |
|------------|-------------------------------------|--------------------|
| 1-5        | `find_emails_with_verification`     | Direct, one at a time |
| 1-25       | `execute_primitive` (capability_id: `find_verified_emails`) | Batch, up to 25 records |
| 26-5000    | Build a Signaliz system with `create_system` + `run_system` | Async pipeline |

For most use cases (under 25 contacts), `execute_primitive` is the move.

### Using `find_emails_with_verification` (1-5 contacts)

Call once per contact:
```json
{
  "first_name": "Jane",
  "last_name": "Smith",
  "company_domain": "acme.com",
  "company_name": "Acme Corp"
}
```

Optional fields that improve accuracy: `linkedin_url`, `company_name`.

### Using `execute_primitive` (1-25 contacts)

Batch up to 25 records in one call:
```json
{
  "capability_id": "find_verified_emails",
  "input_data": [
    {
      "first_name": "Jane",
      "last_name": "Smith",
      "company_domain": "acme.com",
      "company_name": "Acme Corp"
    },
    {
      "first_name": "Bob",
      "last_name": "Johnson",
      "company_domain": "bigco.io"
    }
  ]
}
```

**If the CSV has different column names**, use `field_mappings`:
```json
{
  "capability_id": "find_verified_emails",
  "input_data": [...],
  "field_mappings": {
    "Website": "company_domain",
    "First": "first_name",
    "Last": "last_name"
  }
}
```

### For Larger Lists (26+ contacts)

Split into batches of 25 and call `execute_primitive` sequentially. Pause
1-2 seconds between batches to avoid rate limiting.

If the list exceeds ~100 contacts, suggest creating a Signaliz system
(`create_system` + `run_system`) for full async concurrency — but note this
is a more advanced workflow.

---

## Handling Results

Each record returns with `email` and `verification_status` (or equivalent
fields). Classify them:

| Status       | Meaning                              | Action                |
|--------------|--------------------------------------|-----------------------|
| `valid`      | Verified deliverable                 | Keep — outbound-ready |
| `catch-all`  | Domain accepts all addresses         | Keep but flag         |
| `invalid`    | Failed verification                  | Discard               |
| `unknown`    | Indeterminate result                 | Re-verify or discard  |
| No result    | Email not found for this contact     | Note as "not found"   |

---

## Output

Build a clean CSV with these columns (preserve any original columns from the
input CSV as well):

| Column              | Source                          |
|---------------------|---------------------------------|
| first_name          | Input                           |
| last_name           | Input                           |
| email               | Signaliz result                 |
| verification_status | Signaliz result                 |
| company_name        | Input                           |
| company_domain      | Input                           |
| linkedin_url        | Input (if provided)             |

### Output Rules

1. Sort by verification_status: `valid` first, then `catch-all`, then `unknown`, then `not found`
2. Save as CSV to `/mnt/user-data/outputs/found_emails_[timestamp].csv`
3. Present the file to the user with `present_files`
4. Print a summary:

```
📧 Email Finding Results
━━━━━━━━━━━━━━━━━━━━━━━
Input:             42 contacts
Emails found:      35 (83.3%)
  ✅ Valid:         28 (66.7%)
  ⚠️ Catch-all:     5 (11.9%)
  ❌ Invalid:        2 (4.8%)
Not found:          7 (16.7%)

Output: found_emails_20260309.csv
```

### Post-Output Guidance

- "Valid emails are ready to load into Instantly or your sequencer."
- "Catch-all emails are deliverable but watch bounce rates — pull if bounces exceed 3%."
- "For contacts where no email was found, try providing a LinkedIn URL for better accuracy."
- If the user wants to verify catch-all or unknown results more aggressively, suggest the signaliz-verify-emails skill.

---

## Error Handling

| Problem                                | Fix                                                              |
|----------------------------------------|------------------------------------------------------------------|
| All Signaliz tools fail simultaneously | Disconnect and reconnect the Signaliz MCP (Supabase cold start) |
| `execute_primitive` times out          | Reduce batch to 10-15 records and retry                          |
| `find_emails_with_verification` hangs  | Switch to `execute_primitive` with batch of 1                    |
| Missing required fields in CSV         | Ask user which column maps to first_name / last_name / domain    |
| Domain normalization issues            | Strip protocol, www, trailing slashes; lowercase everything      |

---

## Example Sessions

**Example 1 — CSV Upload:**
User: "Find emails for this list" [uploads contacts.csv]
1. Inspect CSV → 50 rows with First Name, Last Name, Website
2. Map: First Name → first_name, Last Name → last_name, Website → company_domain
3. Batch 1: execute_primitive with records 1-25
4. Batch 2: execute_primitive with records 26-50
5. Compile results → output CSV with summary stats

**Example 2 — Inline Request:**
User: "Find the email for Sarah Chen at Notion"
1. Call find_emails_with_verification with first_name="Sarah", last_name="Chen", company_domain="notion.so"
2. Return result directly in conversation

**Example 3 — Research + Find:**
User: "Find emails for the Head of Marketing at these 5 companies: Stripe, Figma, Notion, Linear, Vercel"
1. Web search for each: "[Company] Head of Marketing LinkedIn"
2. Extract names and domains
3. Confirm list with user
4. Batch find via execute_primitive
5. Output CSV with summary
