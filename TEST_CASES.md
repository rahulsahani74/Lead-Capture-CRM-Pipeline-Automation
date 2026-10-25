# Test Cases

Use the `Send-Lead` PowerShell helper from [SETUP_GUIDE.md](SETUP_GUIDE.md) step 6. For test URLs, click **Execute workflow** in workflow 01 before **each** request. For production URLs, activate the workflow and add `-Url http://localhost:5678/webhook/lead-capture`.

**Status legend:** *Logic-tested* = the Code-node logic was exercised with simulated inputs (37 assertions, all passing). *Needs live test* = depends on real Google Sheets/Telegram/n8n execution and has **not** been run by the author. Fill the last column when you run it.

## Reusable payloads

```powershell
$valid   = '{"name":"Amit Sharma","email":"amit@example.com","phone":"9876543210","company":"ABC Solutions","requirement":"Website Development","source":"Website"}'
$sameEml = '{"name":"Amit S","email":"  AMIT@Example.com ","phone":"9876543210","source":"Google Ads","requirement":"SEO"}'
$phoneOnly = '{"name":"Priya","phone":"98765 43210","source":"WhatsApp"}'
$newPhone  = '{"name":"Ravi","phone":"9123456789","source":"Referral"}'
$noContact = '{"name":"Nobody"}'
$badEmail  = '{"name":"Test","email":"not-an-email"}'
$noName    = '{"email":"x@example.com"}'
$extra     = '{"name":"Extra Fields","email":"extra@example.com","utm_campaign":"diwali","budget":"50000"}'
```

## Matrix

| # | Scenario | Steps | Expected result | Verified by logic test | Live result |
|---|---|---|---|---|---|
| 1 | Valid input, new lead | `Send-Lead $valid` | HTTP 200, `{"success":true,...,"action":"created"}`; new row with Lead ID, Status `New`, Created At and Follow-up At (+24h) in `+05:30`; Telegram "New lead" | Yes (decision + fields) | |
| 2 | Invalid: missing name | `Send-Lead $noName` | HTTP 400, `success:false`, errors include "name is required"; no sheet write; no Telegram | Yes | |
| 3 | Invalid: bad email | `Send-Lead $badEmail` | HTTP 400, "email format is invalid" | Yes | |
| 4 | Invalid: no email and no phone | `Send-Lead $noContact` | HTTP 400, "Provide at least one of email or phone" | Yes | |
| 5 | Invalid: malformed JSON | `Send-Lead '{"name": '` | n8n rejects with HTTP 400 before the workflow runs (or the workflow returns 400); no sheet write | Partly (string body → 400) | |
| 6 | Invalid: empty body | `Send-Lead ''` | HTTP 400, no sheet write | Yes (missing body) | |
| 7 | Duplicate email (case/space differences) | send `$valid`, then `$sameEml` | Second: `action":"updated"`; same row; Lead ID, Status, Source (`Website`), Created At unchanged; Notes gets "Repeat inquiry via Google Ads: SEO" | Yes | |
| 8 | Existing lead keeps sales status | After #1 set Status to `Contacted`, send `$valid` again | Status stays `Contacted`; no new Telegram (repeat notification is disabled by default) | Yes (Status never written) | |
| 9 | Duplicate phone, no email | after #1 send `$phoneOnly` (same digits as the row from #1, no email supplied) | `action":"updated"`, same row as #1 (phone is the secondary key when email is absent); Notes gets a repeat-inquiry line from source `WhatsApp`. Also send `$newPhone` → `created` | Yes (phone match rules) | |
| 10 | Phone formats not equal | send `{"name":"A","phone":"+919876543210"}` after #1 | Treated as a different phone (`+91…` ≠ `9876543210`) → `created`; documented limitation, no country code assumed | Yes | |
| 11 | Extra fields preserved | `Send-Lead $extra` | Row Notes contains `Extra: utm_campaign=diwali; budget=50000` | Yes | |
| 12 | Same lead submitted twice quickly (sequential) | send `$valid` twice, a few seconds apart | One row only | Yes | |
| 13 | Race: same new email, two simultaneous requests | Run two `Send-Lead` in parallel (`Start-Job`) | May create two rows (documented Sheets limitation) | n/a | |
| 14 | Failed Telegram notification | Set a wrong chat ID in Config, send a new lead | Lead row is saved; execution shows the `Notification Failed (lead saved)` path; HTTP 200 created | No | |
| 15 | Failed Sheets write | Temporarily set `sheetName` to a non-existent tab (or revoke the credential), send a lead | HTTP 500 `{"success":false,"message":"Unable to process your request right now..."}`; admin Telegram alert; **no** success response; no internal error text in the response | No | |
| 16 | Due reminder | Lead with Status `New`, Follow-up At in the past, Reminder Sent empty; run workflow 02 manually | One Telegram reminder; Follow-up Reminder Sent filled with `+05:30` timestamp | Yes (selection) / No (send+update) | |
| 17 | Repeated scheduler execution | Run workflow 02 again right after #16 | No reminder (row skipped) | Yes | |
| 18 | Not due / wrong status | Follow-up At in the future, or Status `Contacted` | Skipped | Yes | |
| 19 | Invalid or missing Follow-up At | Blank, or `tomorrow` | Skipped safely, no error | Yes | |
| 20 | Burst cap | 40 due leads | Max 15 reminders per run, the rest next run | Yes | |
| 21 | Reminder update failure | Revoke sheet access after the Telegram step | Admin alert; reminder may repeat next run (documented) | No | |
| 22 | Error workflow | Activate workflow 03, set it as Error Workflow on 01, force an unhandled error (e.g. temporarily put `throw new Error('x')` in `Match & Decide`) and send a lead through the **production** URL | Telegram alert with workflow name, node and execution ID, no lead data | Message builder: Yes; trigger: No | |
| 23 | Header auth (if enabled) | Send without the header, then with | 403 without, 200 with | No | |
| 24 | Demo form | Open `04_demo_lead_form.html`, submit valid and invalid data | Success message; validation messages; errors shown as plain text | No | |

## How to verify rows quickly

After each test open the sheet and check: row count, `Lead ID`/`Created At` unchanged on updates, `Status` untouched, `Notes` content. In n8n open **Executions** and confirm the path taken (Append vs Update branch).
