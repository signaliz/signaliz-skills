---
name: signaliz-verify-emails
description: >
  Verify the deliverability of email addresses using Signaliz MCP. Use this
  skill whenever someone asks to: verify emails, check if emails are valid,
  test deliverability, validate an email list, check bounce risk, scrub emails
  before sending, audit email quality, or determine which emails on a list are
  safe to send to. Trigger on phrases like "verify these emails", "are these
  emails valid", "check deliverability", "which of these can I send to",
  "scrub this email list", "validate these addresses", "check these emails
  before I send", "email verification", "test if this email works", "is this
  email valid", or any request to assess whether existing email addresses are
  deliverable. Also trigger when someone uploads a CSV of emails and wants to
  know which ones are good. Always use this skill when the user already HAS
  email addresses and wants to check them — if they need to FIND emails from
  names, use signaliz-find-verified-emails instead.
---

# Verify Emails via Signaliz

Take a list of email addresses (from CSV, pasted inline, or from another
workflow) and verify their deliverability using Signaliz's email verification
engine. Returns per-email status (valid, invalid, catch-all, unknown) so the
user knows exactly which addresses are safe to send to.

---

## Before You Start

**Required:** The Signaliz MCP must be connected. If tools like
`verify_emails` or `execute_primitive` aren't responding, tell the user to
disconnect and reconnect the Signaliz MCP in their Claude settings (Supabase
edge function cold starts can cause all tools to fail at once — a reconnect
fixes it).

**Tool loading:** Before making any Signaliz MCP calls, use `tool_search` to
load the correct tool definitions. Do not guess parameter names.

---

## Input Modes

### Mode A: CSV Upload

The user uploads a CSV containing email addresses. Inspect it and find the
email column:

| Required Field | Common CSV Variants                                         |
|----------------|-------------------------------------------------------------|
| `email`        | Email, email_address, Email Address, work_email, E-mail     |

Also look for supplementary columns that should be preserved in output:
first_name, last_name, company_name, company_domain, title, etc.

**Pre-processing before verification:**
1. Normalize: lowercase all emails, trim whitespace
2. Deduplicate: remove duplicate email addresses (keep first occurrence, log count)
3. Remove obvious invalids: empty values, missing @ sign, no domain portion
4. Strip mailto: prefixes if present

Report what you found:
```
Found 200 emails in column "Email Address"
Removed 8 duplicates, 3 empty rows, 1 malformed address
188 unique emails ready for verification
```

### Mode B: Inline / Conversational Input

The user provides emails directly in conversation — either a short list
pasted in, a single email to check, or emails that came from a previous
workflow step. Collect them and proceed.

For a single email, just verify it and return the result conversationally
(no need for CSV output).

---

## Execution: Verifying Emails

### Choosing the Right Tool

| Batch Size | Tool                                | Pattern              |
|------------|-------------------------------------|----------------------|
| 1          | `execute_primitive` (capability: `verify_emails`) | Instant result       |
| 2-25       | `execute_primitive` (capability: `verify_emails`) | Batch, instant       |
| 26-5000    | `Signaliz:verify_emails`            | Async job + polling  |

### Small Batches (1-25 emails): `execute_primitive`

Use `execute_primitive` with `capability_id: "verify_emails"`:

```json
{
  "capability_id": "verify_emails",
  "input_data": [
    {"email": "jane@acme.com"},
    {"email": "bob@bigco.io"},
    {"email": "alice@startup.co"}
  ]
}
```

Results come back immediately with verification status per email.

**If the CSV column isn't named `email`**, use `field_mappings`:
```json
{
  "capability_id": "verify_emails",
  "input_data": [...],
  "field_mappings": {
    "Email Address": "email"
  }
}
```

### Large Batches (26-5000 emails): `Signaliz:verify_emails`

Use the dedicated batch tool:

```json
{
  "emails": [
    {"email": "jane@acme.com"},
    {"email": "bob@bigco.io"},
    ...
  ]
}
```

This returns a `job_id` immediately. You must poll for results.

### Polling for Results

After getting a `job_id` from `verify_emails`, poll with `check_job_status`:

1. **First poll** — wait 5-10 seconds, then call:
   ```json
   {"job_id": "<job_id>", "_poll_count": 1, "page_size": 500}
   ```
2. **Check status** — if `status` is `processing` or `queued`, wait the
   number of seconds indicated by `next_poll_after_seconds` in the response
3. **Repeat** — increment `_poll_count` each time. The tool uses exponential
   backoff via this counter
4. **When `status` = `completed`** — results are in the response. If there
   are more pages, paginate with `page` parameter
5. **Pagination** — use `page_size: 500` and increment `page` until you've
   retrieved all records

**Known quirk:** In some environments, `check_job_status` caps displayed
results at 5 per call regardless of `page_size`. If this happens, paginate
with `page_size: 5` and work through all pages.

---

## Interpreting Results

Map verification statuses to actionable categories:

| Signaliz Status               | Category        | Safe to Send? | Recommendation                    |
|-------------------------------|-----------------|---------------|-----------------------------------|
| `valid` / `deliverable`       | Valid           | Yes           | Outbound-ready                    |
| `catch_all_deliverable`       | Catch-all Safe  | Yes (careful) | Send, but monitor bounce rates    |
| `catch-all` / `catch_all`     | Catch-all       | Risky         | Send at low volume, watch bounces |
| `risky`                       | Risky           | No            | Remove or re-verify later         |
| `invalid` / `undeliverable`   | Invalid         | No            | Remove from list immediately      |
| `unknown`                     | Unknown         | No            | Remove or re-verify with LinkedIn |

**Status field names may vary slightly between tools** (`verification_status`,
`status`, `result`). Check the actual response fields.

---

## Output

### For Single Email Verification

Return the result conversationally:
```
✅ jane@acme.com — Valid (deliverable, not catch-all)
```
or
```
❌ bob@fakeco.xyz — Invalid (mailbox does not exist)
```

### For Batch Verification (CSV output)

Build a CSV that includes all original columns plus verification results:

| Column              | Source                          |
|---------------------|---------------------------------|
| email               | Input                           |
| verification_status | Signaliz result                 |
| catch_all           | Signaliz result (boolean)       |
| (all other columns) | Preserved from input            |

Write three output files:

1. **`verified_clean_[filename].csv`** — Only valid and catch-all-deliverable
   emails (safe to send)
2. **`verified_removed_[filename].csv`** — Invalid, unknown, and risky emails
   (for review)
3. **`verified_full_[filename].csv`** — All emails with status appended
   (full audit trail)

Save to `/mnt/user-data/outputs/` and present with `present_files`.

### Summary Stats

Always print a summary after verification:

```
📬 Email Verification Results
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Input:              188 unique emails
Duplicates removed:   8

Verification Results:
  ✅ Valid:            121 (64.4%)
  ⚠️ Catch-all:        34 (18.1%)
  ❌ Invalid:           22 (11.7%)
  🔍 Unknown:           11 (5.9%)

Safe to send:       155 emails → verified_clean_contacts.csv
Removed:             33 emails → verified_removed_contacts.csv
Full audit:         188 emails → verified_full_contacts.csv
```

### Post-Output Guidance

- "Valid emails are ready for sequencing in Instantly or your outbound tool."
- "Catch-all domains accept all addresses — deliverable but can't confirm the specific mailbox exists. Monitor bounce rates and pull if bounces exceed 3%."
- "Invalid emails should never be sent to — they'll hard bounce and damage sender reputation."
- "For unknown results, providing a LinkedIn URL and re-running through the email finder can sometimes resolve them."

---

## Error Handling

| Problem                                  | Fix                                                              |
|------------------------------------------|------------------------------------------------------------------|
| All Signaliz tools fail simultaneously   | Disconnect and reconnect the Signaliz MCP (Supabase cold start)  |
| `execute_primitive` times out            | Reduce batch to 10-15 records and retry                           |
| `verify_emails` job stuck at "queued"    | Wait 30s and re-poll. If stuck >2min, resubmit the job            |
| `check_job_status` returns max 5 results | Paginate with page_size=5 through all pages                       |
| No email column found in CSV             | Ask the user which column contains the email addresses            |
| Mixed formats (some with mailto:, etc.)  | Normalize before submission — strip prefixes, lowercase, trim     |

---

## Example Sessions

**Example 1 — CSV Upload:**
User: "Verify these emails" [uploads email_list.csv]
1. Inspect CSV → 300 rows, email column = "Email Address"
2. Normalize, dedupe → 285 unique emails
3. Call `Signaliz:verify_emails` (batch endpoint, async)
4. Poll `check_job_status` until complete
5. Classify results → output 3 CSVs with summary

**Example 2 — Quick Single Check:**
User: "Is josh@signaliz.com a valid email?"
1. Call `execute_primitive` with capability_id="verify_emails", input_data=[{"email": "josh@signaliz.com"}]
2. Return result conversationally

**Example 3 — Inline List:**
User: "Check these emails: alice@stripe.com, bob@fakeco.xyz, carol@notion.so"
1. Call `execute_primitive` with all 3 emails
2. Return results in a table format in conversation
3. Offer to export as CSV if desired

**Example 4 — Post-Finding Verification:**
User: "I just found some emails — can you verify the catch-all ones more carefully?"
1. Extract catch-all emails from previous results
2. Re-verify via `execute_primitive`
3. Update classification
