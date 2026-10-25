# Google Sheets Schema

Create one spreadsheet with a tab named exactly **`Leads`**.

## Row 1 headers (exact spelling, case and hyphens matter)

Paste this into cell A1 (tab-separated) or type each header in its own cell, A to N:

```
Lead ID	Name	Email	Phone	Company	Requirement	Source	Status	Assigned To	Created At	Follow-up At	Last Contacted	Notes	Follow-up Reminder Sent
```

| Col | Header | Written by | Notes |
|---|---|---|---|
| A | Lead ID | Workflow 1 | Generated once (`LD-YYYYMMDD-XXXXXX`), never changed on updates |
| B | Name | Workflow 1 | Filled on create; on a repeat inquiry only filled if empty |
| C | Email | Workflow 1 | Lowercased. Primary duplicate key |
| D | Phone | Workflow 1 | Digits, leading `+` kept if supplied. No country code is ever added. Secondary duplicate key |
| E | Company | Workflow 1 | Optional |
| F | Requirement | Workflow 1 | Optional. Later requirements from repeat inquiries go into Notes |
| G | Source | Workflow 1 | Original source preserved forever (default `Unknown`) |
| H | Status | Workflow 1 creates as `New`; **you** edit it | Workflows never reset it. Reminders only go out while it is `New` |
| I | Assigned To | You | Free text |
| J | Created At | Workflow 1 | ISO 8601 with `+05:30`, e.g. `2026-10-25T13:50:00+05:30` |
| K | Follow-up At | Workflow 1 | Created At + `followUpHours` (default 24). Must stay ISO 8601 with offset |
| L | Last Contacted | You | Free text/date |
| M | Notes | Workflow 1 appends | Repeat inquiries, extra form fields, "Review" flags |
| N | Follow-up Reminder Sent | Workflow 2 | ISO timestamp once the reminder was sent. Empty = not yet reminded |

## Formatting rules (important)

1. Select columns **A:N → Format → Number → Plain text**. The workflows write with `RAW` mode, so values are stored as typed, but Plain text stops Sheets from re-interpreting values if someone edits a cell by hand (a changed `Follow-up At` that Sheets turns into a locale date will be skipped by the scheduler as an invalid date).
2. Do not insert columns between A and N, rename headers, or leave blank header cells.
3. You may add extra columns to the right of N; the workflows ignore them.
4. Keep data starting at row 2 with no blank rows in the middle.

## Duplicate matching rules (Workflow 1)

1. **Email present** → compare lowercased/trimmed email with the Email column. Match found → update that row (the lowest row number if the sheet already holds duplicates).
2. Email present but no email match → look for **one** row that has an **empty Email** and the same phone digits; if exactly one, it is enriched with the new email. If several, nothing is merged and a new row is created with a `Review:` note.
3. **No email** → match on phone digits. Exactly one match → update. Several matches → ambiguous: new row plus `Review:` note.
4. Phones are compared as digits only. `9876543210` and `+919876543210` are **different** keys, because assuming a country code would be wrong for some leads. Submit phones consistently, or normalise old rows.
5. An email match is never overridden by a phone match: the same phone with a *different* email on a row that already has an email is treated as a new lead (shared office/family lines).

On a repeat inquiry the workflow writes only Name/Email/Phone/Company/Requirement (existing values kept; blanks filled) and appends a line to Notes. It never touches Lead ID, Status, Source, Created At, Follow-up At, Assigned To, Last Contacted or Follow-up Reminder Sent.

## Concurrency and race-condition limits

Google Sheets is not a database. Workflow 1 reads the whole sheet, decides in a Code node, then writes.

- Two submissions of the same new email arriving within roughly the same second can both read "no match" and both append → a duplicate row. n8n executes webhook calls in parallel by default.
- Two parallel updates of the same row can overwrite each other's Notes line.
- Reading the whole sheet on each submission gets slower as rows grow (comfortable up to a few thousand rows; Google API quotas apply, about 60 read requests per minute per user by default).

Mitigations: keep volume low; add a de-duplication step in the sheet (e.g. a weekly cleanup); set `N8N_CONCURRENCY_PRODUCTION_LIMIT=1` if you can accept serialised processing (this affects all workflows); move to Postgres/Airtable/HubSpot with a unique-key constraint when volume grows.

## Access and privacy

The sheet holds personal data. Share it only with the Google account used for the n8n credential and your sales team. Do not publish it to the web.
