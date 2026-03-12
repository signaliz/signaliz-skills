---
name: signaliz-crm-sync-audit
description: >
  Audit, clean, and sync CRM contact data using Signaliz email verification
  and enrichment. Use when someone asks to: audit CRM data, clean CRM
  contacts, verify CRM emails, check for stale or bounced contacts, sync
  enrichment data to their CRM, or maintain CRM hygiene. Trigger on phrases
  like "audit my CRM", "clean my contacts", "verify my CRM emails", "check
  for bad emails in HubSpot", "CRM data quality check", "enrich my CRM
  contacts", "find stale contacts", "CRM hygiene", "check my Salesforce
  contacts", "data decay audit", or any request to assess, clean, or enrich
  existing CRM data. Also trigger when someone exports a CSV from their CRM
  and wants it cleaned and re-imported. Works with HubSpot, Salesforce,
  Pipedrive, or any CRM via CSV. For verifying a standalone email list (not
  from a CRM), use signaliz-verify-emails instead.
metadata:
  author: signaliz
  version: '1.0'
license: MIT
---

# CRM Sync & Audit via Signaliz

Audit CRM contact data for quality issues — stale emails, invalid addresses,
missing fields, duplicate records — then clean and enrich using Signaliz
verification and signal enrichment. Produces actionable reports and clean files
ready for CRM re-import.

---

## Before You Start

**Required:** The Signaliz MCP must be connected. If tools like
`execute_primitive` or `verify_emails` aren't responding, tell the user to
disconnect and reconnect the Signaliz MCP in their settings (Supabase edge
function cold starts can cause all tools to fail at once — a reconnect
fixes it).

**Tool loading:** Before making any Signaliz MCP calls, use `tool_search` to
load the correct tool definitions. Do not guess parameter names.

**CRM access:** This skill works with CSV exports from any CRM. If the user
has a direct CRM MCP connection (HubSpot, Salesforce, Pipedrive), use it to
pull contacts directly. Otherwise, ask the user to export a CSV from their CRM.

---

## Input Modes

### Mode A: CSV Export from CRM

The user exports a CSV from their CRM and uploads it. Inspect the file and
map columns:

| Required Field   | Common CRM Export Variants                               |
|------------------|----------------------------------------------------------|
| `email`          | Email, Email Address, Work Email, Contact Email          |

| Helpful Fields   | Common CRM Export Variants                               |
|------------------|----------------------------------------------------------|
| `first_name`     | First Name, FirstName, first, Contact First Name         |
| `last_name`      | Last Name, LastName, last, Contact Last Name             |
| `company_name`   | Company, Company Name, Account Name, Organization        |
| `company_domain` | Domain, Website, Company Domain, Company Website         |
| `job_title`      | Title, Job Title, Role, Position                         |
| `phone`          | Phone, Phone Number, Mobile, Work Phone                  |
| `owner`          | Contact Owner, Owner, Assigned To, Sales Rep             |
| `created_date`   | Created Date, Date Added, Created At                     |
| `last_activity`  | Last Activity, Last Activity Date, Last Modified         |
| `deal_stage`     | Deal Stage, Pipeline Stage, Opportunity Stage            |
| `lead_status`    | Status, Lead Status, Contact Status                      |

Report what you found:
```
CRM Export Analysis:
  Records:         2,450 contacts
  Email column:    "Email Address"
  Company column:  "Company Name"
  Owner column:    "Contact Owner"
  Created range:   Jan 2023 — Mar 2026

  Pre-audit findings:
    Empty emails:    47 records (1.9%)
    Duplicate emails: 23 records (0.9%)
    Missing company:  89 records (3.6%)
```

### Mode B: Direct CRM Connection

If the user has a CRM MCP connected (HubSpot, Salesforce, etc.), pull contacts
directly:

1. Query the CRM for all contacts (or a filtered segment the user specifies)
2. Extract relevant fields
3. Proceed with the audit workflow

### Mode C: Targeted Segment

The user wants to audit a specific segment:
- "Check all contacts added in the last 6 months"
- "Verify emails for contacts in the 'Nurture' pipeline stage"
- "Audit contacts owned by [rep name]"

Filter the dataset before running the full audit.

---

## Audit Workflow

### Phase 1: Data Quality Assessment

Before any API calls, analyze the raw data for issues:

**1.1 Completeness check:**
- Missing email addresses
- Missing first/last name
- Missing company name/domain
- Missing job title
- Missing phone number

**1.2 Duplicate detection:**
- Exact email duplicates
- Fuzzy name + company matches (same person, different email)
- Same person across multiple company domains (job changers)

**1.3 Format validation:**
- Malformed email syntax (missing @, invalid characters)
- Free email domains (gmail.com, yahoo.com, hotmail.com) — flag, don't remove
- Role-based emails (info@, sales@, support@) — flag as low-value
- Disposable email domains — flag for removal

**1.4 Staleness indicators:**
- Contacts with no activity in 6+ months
- Created date vs. last activity (engagement decay)
- Companies that may have been acquired or shut down

Present the pre-verification assessment:

```
📋 Data Quality Assessment
━━━━━━━━━━━━━━━━━━━━━━━━━━
Total records:          2,450

Completeness:
  ✅ Has email:          2,403 (98.1%)
  ⚠️ Missing email:        47 (1.9%)
  ⚠️ Missing company:      89 (3.6%)
  ⚠️ Missing title:       156 (6.4%)

Duplicates:
  🔁 Exact email dupes:    23 (0.9%)
  🔁 Fuzzy match dupes:    11 (0.4%)

Format issues:
  ❌ Malformed emails:       8 (0.3%)
  📧 Free email domains:    67 (2.7%)
  📧 Role-based emails:     34 (1.4%)

Staleness:
  💤 No activity 6+ months: 412 (16.8%)
  💤 No activity 12+ months: 189 (7.7%)

Recommendation: Verify 2,403 emails via Signaliz (est. ~2 minutes)
```

Ask the user to confirm before proceeding to verification.

---

### Phase 2: Email Verification (Signaliz)

Verify all emails using Signaliz:

**For 1-25 emails** — use `execute_primitive`:
```json
{
  "capability_id": "verify_emails",
  "input_data": [
    {"email": "jane@acme.com"},
    {"email": "bob@bigco.io"}
  ]
}
```

**For 26-5,000 emails** — use `verify_emails` (async):
```json
{
  "emails": [
    {"email": "jane@acme.com"},
    {"email": "bob@bigco.io"}
  ]
}
```

Poll with `check_job_status` until complete. Paginate results with
`page_size: 500`.

**Classify results:**

| Status                        | Category          | CRM Action                      |
|-------------------------------|-------------------|---------------------------------|
| `valid` / `deliverable`      | Valid             | Keep — no action needed         |
| `catch_all_deliverable`      | Catch-all (safe)  | Keep — add "catch-all" tag      |
| `catch-all` / `catch_all`    | Catch-all (risky) | Keep but flag — monitor bounces |
| `risky`                      | Risky             | Move to quarantine list         |
| `invalid` / `undeliverable`  | Invalid           | Remove or archive               |
| `unknown`                    | Unknown           | Re-verify in 30 days            |

---

### Phase 3: Enrichment Gap Analysis

For contacts that are valid but have missing fields, assess what Signaliz
can fill in:

**Missing company domain:** If the contact has a company name but no domain,
attempt to derive the domain from the email address (everything after @).

**Missing signals:** For contacts with company domains, optionally run
`execute_primitive` with `enrich_company_signals` to check for recent
signals at their companies. This helps identify:
- Contacts at companies that are actively hiring (growth signal)
- Contacts at companies with recent funding (budget signal)
- Contacts at companies with leadership changes (buying window)

**Email re-finding:** For contacts where the email is now invalid (they may
have changed jobs), attempt to find their new email using
`execute_primitive` with `find_verified_emails` if name + new company domain
is discoverable.

---

### Phase 4: Output Files

Generate three output files:

#### File 1: Clean Contacts (CRM re-import ready)

All contacts with valid or catch-all-safe emails, duplicates removed, fields
normalized:

| Column              | Content                                    |
|---------------------|--------------------------------------------|
| (all original cols) | Preserved from CRM export                  |
| email_status        | valid / catch-all / catch-all-safe         |
| verification_date   | Today's date                               |
| signaliz_verified   | TRUE                                       |

#### File 2: Removed / Quarantined Contacts

Contacts removed from the clean file, with reasons:

| Column              | Content                                    |
|---------------------|--------------------------------------------|
| (all original cols) | Preserved from CRM export                  |
| email_status        | invalid / risky / unknown                  |
| removal_reason      | Bounced / Invalid / Duplicate / Malformed  |
| recommendation      | Delete / Archive / Re-verify in 30 days    |

#### File 3: Audit Report Summary

A structured summary with actionable recommendations.

---

## Audit Report

Present the complete audit results:

```
📊 CRM Data Quality Audit Report
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
CRM source:            HubSpot export — All Contacts
Audit date:            2026-03-11
Total records:         2,450

EMAIL VERIFICATION RESULTS
━━━━━━━━━━━━━━━━━━━━━━━━━━
  ✅ Valid:              1,847 (75.4%)
  ⚠️ Catch-all (safe):    198 (8.1%)
  ⚠️ Catch-all (risky):    89 (3.6%)
  ❌ Invalid:              187 (7.6%)
  ❓ Unknown:               82 (3.3%)
  📭 Missing email:         47 (1.9%)

DATA QUALITY ISSUES
━━━━━━━━━━━━━━━━━━━
  🔁 Duplicates removed:    34
  ❌ Malformed emails:        8
  📧 Free email domains:     67
  📧 Role-based emails:      34
  💤 Stale (no activity 6mo+): 412

HEALTH SCORE: 75.4% clean
━━━━━━━━━━━━━━━━━━━━━━━━━
Industry benchmark: 70-80% is typical for a 3-year-old CRM

RECOMMENDATIONS
━━━━━━━━━━━━━━━
1. IMMEDIATE: Remove 187 invalid emails to protect sender reputation
2. IMMEDIATE: Merge 34 duplicate records (keep most recent activity)
3. SHORT-TERM: Re-engage or archive 412 stale contacts
4. SHORT-TERM: Re-verify 82 unknown emails in 30 days
5. ONGOING: Verify new contacts at import to prevent future decay
6. ONGOING: Run this audit quarterly to maintain data quality

ESTIMATED IMPACT
━━━━━━━━━━━━━━━━
  Bounce rate reduction:     ~7.6% → <1% (if invalids removed)
  Deliverability improvement: +12-15% (estimated)
  CRM record count:          2,450 → 2,229 (clean, actionable)

OUTPUT FILES
━━━━━━━━━━━━
  📄 crm_clean_contacts_20260311.csv — 2,045 verified contacts (re-import)
  📄 crm_removed_contacts_20260311.csv — 187 invalid + 34 dupes (review)
  📄 crm_audit_report_20260311.csv — Full audit trail with statuses
```

---

## Post-Audit Guidance

After presenting the audit:

- "Import crm_clean_contacts to your CRM — these are verified and safe to email."
- "Review crm_removed_contacts before deleting — some may be recoverable if the contact changed companies."
- "Want me to try finding new emails for the 187 invalid contacts? I can search by name + company." → signaliz-find-verified-emails
- "Want me to enrich the valid contacts with company signals to find the best outreach candidates?" → signaliz-company-signals
- "I recommend running this audit quarterly. Want me to set that up?"

### Re-Import Guidance by CRM

**HubSpot:**
1. Go to Contacts → Import
2. Upload crm_clean_contacts CSV
3. Map the `email_status` and `signaliz_verified` fields to custom properties
4. Choose "Update existing contacts" to overwrite stale data

**Salesforce:**
1. Use Data Import Wizard or Data Loader
2. Match on Email field to update existing records
3. Map verification fields to custom fields

**Pipedrive:**
1. Go to Contacts → Import from spreadsheet
2. Upload the clean CSV
3. Match on email to update existing contacts

---

## Error Handling

| Problem                                | Fix                                                              |
|----------------------------------------|------------------------------------------------------------------|
| All Signaliz tools fail simultaneously | Disconnect and reconnect the Signaliz MCP (Supabase cold start)  |
| CSV too large (>5,000 rows)            | Split into batches of 5,000 and process sequentially             |
| `verify_emails` job stuck              | Wait 60s and re-poll. If stuck >3min, resubmit                   |
| CRM export has unexpected encoding     | Try reading as UTF-8, then Latin-1; normalize before processing  |
| Duplicate detection false positives    | Present potential dupes to user for manual review                |
| Missing email column                   | Ask user which column contains email addresses                   |

---

## Example Sessions

**Example 1 — Full CRM Audit:**
User: "Audit my HubSpot contacts — here's the export" [uploads hubspot_contacts.csv]
1. Inspect CSV → 2,450 records, map columns
2. Phase 1: Data quality assessment → report findings
3. User confirms → proceed with verification
4. Phase 2: Verify 2,403 emails via Signaliz (async job)
5. Phase 3: Identify enrichment gaps
6. Phase 4: Generate 3 output files
7. Present audit report with recommendations

**Example 2 — Targeted Segment Audit:**
User: "Check if the emails are still valid for contacts in my 'Cold Outreach' pipeline"
1. User exports the segment (or pull via CRM MCP)
2. Run verification on the segment only
3. Report which contacts have decayed
4. Offer to find replacement emails for invalid contacts

**Example 3 — Pre-Campaign Hygiene:**
User: "I'm about to send a campaign to 500 contacts — can you verify them first?"
1. User provides the list
2. Fast-track verification (skip full audit, focus on email status)
3. Split into "safe to send" and "do not send" lists
4. Flag catch-all domains with bounce risk warnings

**Example 4 — Quarterly Maintenance:**
User: "Run our quarterly CRM audit"
1. User exports full CRM
2. Full audit workflow (Phases 1-4)
3. Compare to previous audit if available (track decay rate over time)
4. Generate trend report: "Email decay rate: 2.1% per quarter"
