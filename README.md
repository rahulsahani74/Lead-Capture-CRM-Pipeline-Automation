# Lead Capture & CRM Pipeline Automation (n8n)

Importable n8n workflows that capture website leads through a webhook, validate and normalise them, detect duplicates, store them in Google Sheets, notify your team on Telegram, and send follow-up reminders.

Built for: self-hosted n8n Community Edition (Docker Desktop on Windows), Google Sheets as the database, Telegram for notifications, time zone `Asia/Kolkata`. Zero-cost; only built-in n8n nodes.

## What's in the box

| File | Purpose |
|---|---|
| `01_lead_capture_crm.json` | **Workflow 1: Lead Capture and CRM Sync** |
| `02_follow_up_scheduler.json` | **Workflow 2: Follow-up Scheduler** (hourly) |
| `03_error_monitoring.json` | **Workflow 3: Error Monitoring** (Error Trigger → Telegram) |
| `04_demo_lead_form.html` | Self-contained demo form that POSTs JSON to the webhook |
| `SETUP_GUIDE.md` | Step-by-step setup, PowerShell tests, troubleshooting, backups, production checklist |
| `GOOGLE_SHEETS_SCHEMA.md` | Exact sheet headers, matching rules, concurrency limits |
| `TEST_CASES.md` | Test matrix and copy-paste payloads |
| `OPTIONAL_INTEGRATIONS.md` | HubSpot, Airtable, email, WhatsApp, Google Ads, Resend notes |
| `.gitignore` | Keeps secrets and local n8n data out of Git |

## How it works

**Workflow 1**

```
Lead Webhook (POST /lead-capture)
 → Config → Validate & Normalize → Is Valid?
     no  → Respond 400 (list of validation errors)
     yes → Read Leads (Google Sheets) → Match & Decide → Is New Lead?
              new  → Append New Lead → Notify New Lead (Telegram) → Respond 200 {"action":"created"}
              dup  → Update Existing Lead → Notify Repeat Inquiry (disabled by default) → Respond 200 {"action":"updated"}
     Sheets step fails → Alert Admin (Telegram) → Respond 500 (generic message, no internals)
     Telegram fails after the lead is saved → still Respond 200 (lead is safe)
```

- The webhook uses *Respond to Webhook* mode, so the caller only gets an answer after Sheets has been written. A failed write returns HTTP 500, never success.
- Validation: trims text, lowercases email, normalises phone without assuming a country code (keeps `+`, converts leading `00` to `+`), requires a name plus email or phone, enforces length limits, keeps extra scalar fields in Notes, defaults source to `Unknown`, stamps ISO 8601 times in `+05:30`.
- Duplicate detection: email first; phone when email is absent (or to enrich a row that has no email), only when unambiguous. Updates never change Lead ID, Created At, Status, Source or Follow-up fields, and append a "Repeat inquiry" line to Notes. Details in `GOOGLE_SHEETS_SCHEMA.md`.

**Workflow 2**: every hour (Asia/Kolkata) and on manual run: reads the sheet, selects leads with Status `New`, a due and valid `Follow-up At`, and empty `Follow-up Reminder Sent` (max 15 per run), sends a Telegram reminder, and only after that succeeds writes the timestamp to `Follow-up Reminder Sent`. Reminders therefore are not repeated.

**Workflow 3**: Error Trigger → builds a message with workflow name, failed node, mode, execution ID and a redacted, truncated error text → Telegram admin chat. Set it as the *Error Workflow* in the settings of workflows 1 and 2.

## Quick start

1. Start n8n, import the three JSON files (⋯ → Import from file).
2. Create the `Leads` sheet with the exact headers; connect Google Sheets OAuth2 and a Telegram bot credential.
3. Edit the **Config** node in each workflow (spreadsheet ID, Telegram chat IDs).
4. Test with PowerShell (`SETUP_GUIDE.md` step 6), then activate workflows 1, 2 and 3.

Webhook URLs (default n8n on this machine):

- Test: `http://localhost:5678/webhook-test/lead-capture`
- Production (workflow must be Active): `http://localhost:5678/webhook/lead-capture`

Example payload:

```json
{
  "name": "Amit Sharma",
  "email": "amit@example.com",
  "phone": "9876543210",
  "company": "ABC Solutions",
  "requirement": "Website Development",
  "source": "Website"
}
```

## What was and wasn't tested

- **Done:** JSON syntax; node types and `typeVersion`s checked against the `n8n-nodes-base` 2.15.1 package; parameter names for Webhook, Respond to Webhook, If, Set, Code, Google Sheets (v4.7 read/append/update), Telegram, Schedule/Manual/Error triggers read from the package source; connection integrity; no credentials or secrets in files; JavaScript of all Code nodes run against 37 simulated cases (validation, normalisation, duplicate matching, reminder selection, error redaction).
- **Not done:** no execution inside a live n8n with real Google/Telegram accounts. The Google Sheets resource-mapper block (`columns` with `schema`, `matchingColumns: ["row_number"]`) follows n8n's saved-workflow format as I know it and the node source, but could not be confirmed by importing. If n8n shows a Google Sheets node with empty column mappings after import, open the node, re-select the document/sheet and map the columns as listed in the node (field names equal the headers).
- Requires an n8n version that supports Google Sheets node 4.7, Telegram 1.2, Webhook 2.1, Respond to Webhook 1.5, Set 3.4, If 2.2 (all present in n8n-nodes-base 2.15.1).

## Known limitations

- Google Sheets has no transactions: simultaneous submissions of the same new lead can create duplicate rows; reads scan the whole sheet (see `GOOGLE_SHEETS_SCHEMA.md`).
- `9876543210` and `+919876543210` are treated as different phones.
- The webhook ships unauthenticated for easy testing. Enable Header Auth or a proxy before public use (`SETUP_GUIDE.md` step 10).
- A local n8n URL is not reachable from the internet; public traffic needs HTTPS exposure.
- The demo form sends `text/plain` JSON to avoid a CORS preflight, because the Webhook node in this n8n version offers no OPTIONS/CORS setting; behaviour with your particular proxy is unverified.
- If Telegram succeeds but the reminder timestamp update fails, that lead can be reminded again on the next run (an admin alert is sent).
