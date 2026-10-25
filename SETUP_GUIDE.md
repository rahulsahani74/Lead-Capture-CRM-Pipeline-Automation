# Setup Guide (Windows 10/11 + Docker Desktop)

Follow the steps in order. Everything here uses free tools. Estimated time: 30-45 minutes the first time.

> **Verification status.** The workflow files were generated against the node definitions in the `n8n-nodes-base` 2.15.1 package and passed structural checks (valid JSON, real node types/versions, consistent connections, no embedded credentials). The JavaScript logic in the Code nodes was tested with simulated inputs. The workflows have **not** been executed inside a live n8n instance with real Google Sheets/Telegram accounts. Treat your first run as the real test and use [TEST_CASES.md](TEST_CASES.md).

---

## 0. Start n8n (skip if already running)

PowerShell:

```powershell
docker volume create n8n_data
docker run -d --name n8n -p 5678:5678 `
  -e GENERIC_TIMEZONE=Asia/Kolkata -e TZ=Asia/Kolkata `
  -v n8n_data:/home/node/.n8n `
  docker.n8n.io/n8nio/n8n
```

Open http://localhost:5678 and create the owner account. The files were built against n8n-nodes-base 2.15.1; if your n8n is much older, a node may show "version not supported" on import. Update the image (`docker pull docker.n8n.io/n8nio/n8n`) and recreate the container (the `n8n_data` volume keeps your data).

## 1. Import the workflows

For each of `01_lead_capture_crm.json`, `02_follow_up_scheduler.json`, `03_error_monitoring.json`:

1. In n8n click **Create workflow** (or the **+** button).
2. Click the **⋯** menu (top right) → **Import from file…**
3. Choose the JSON file.
4. Save (Ctrl+S).

Credentials will show warning icons: that is expected. They are not included in the files.

## 2. Create the Google spreadsheet

1. Go to https://sheets.google.com → **Blank spreadsheet**. Name it e.g. `Lead CRM`.
2. Rename the first tab to exactly `Leads`.
3. Put the 14 headers in row 1 exactly as listed in [GOOGLE_SHEETS_SCHEMA.md](GOOGLE_SHEETS_SCHEMA.md).
4. Select columns A:N → **Format → Number → Plain text**.
5. Copy the **spreadsheet ID** from the URL: `https://docs.google.com/spreadsheets/d/`**`THIS_PART`**`/edit`.

## 3. Google Sheets OAuth2 credential

1. Open https://console.cloud.google.com → create a project (free).
2. **APIs & Services → Library** → search **Google Sheets API** → **Enable**. (Also enable **Google Drive API** if n8n's credential dialog asks for it.)
3. **APIs & Services → OAuth consent screen**: choose *External*, fill the app name and your email, and under **Test users** add your own Google account. Leave it in *Testing* mode for development.
4. **Credentials → Create credentials → OAuth client ID** → Application type **Web application**. Under *Authorized redirect URIs* add:
   `http://localhost:5678/rest/oauth2-credential/callback`
5. Copy the **Client ID** and **Client Secret**.
6. In n8n: open any Google Sheets node → **Credential to connect with → Create new credential** → *Google Sheets OAuth2 API*. Paste Client ID and Secret, click **Sign in with Google**, approve. n8n requests the scopes it needs automatically.
7. Name it `Google Sheets account` and save.
8. Open each Google Sheets node in workflows 01 and 02 (`Read Leads`, `Append New Lead`, `Update Existing Lead`, `Update Reminder Fields`) and select this credential. (Once selected in one node n8n usually offers it in the others.)

Note: in *Testing* mode Google expires refresh tokens after 7 days; reconnect if the credential stops working, or publish the consent screen for longer-lived tokens.

## 4. Telegram bot and chat ID

1. In Telegram, chat with **@BotFather** → send `/newbot` → choose a name and a username ending in `bot`. Copy the **token** (looks like `123456789:AA…`). Treat it like a password.
2. Send any message to your new bot (or add the bot to your sales group and write a message there).
3. Find the chat ID. In a browser open (replace `<TOKEN>`; do this on your own machine only, never share the URL):
   `https://api.telegram.org/bot<TOKEN>/getUpdates`
   Look for `"chat":{"id":123456789,...}`. Group IDs are negative numbers (e.g. `-1001234567890`). If the result is empty, send another message to the bot and reload.
4. In n8n: open a Telegram node → **Credential → Create new** → *Telegram API* → paste the token (Access Token) → save.
5. Select that credential on every Telegram node (`Notify New Lead`, `Notify Repeat Inquiry (optional)`, `Alert Admin (Sheets Failure)` in workflow 01; `Send Telegram Reminder`, `Alert Admin (Scheduler Failure)` in workflow 02; `Alert Admin` in workflow 03).

## 5. Fill in the `Config` node (in each workflow)

Every workflow starts with a **Config** node. Edit it once per workflow:

| Field | Workflows | Value |
|---|---|---|
| `spreadsheetId` | 01, 02 | the ID from step 2 |
| `sheetName` | 01, 02 | `Leads` (keep unless you renamed the tab) |
| `telegramChatId` | 01, 02 | chat ID for the sales team |
| `adminChatId` | 01, 02, 03 | chat ID for admin alerts (can equal the team chat) |
| `followUpHours` | 01 | hours until the first follow-up is due (default 24) |
| `corsAllowOrigin` | 01 | `*` for testing; your site origin (e.g. `https://example.com`) in production |

Nothing else needs editing. Save each workflow.

## 6. Test workflow 01 before activating

1. Open workflow 01 → click **Execute workflow** (it begins listening on the **Test URL**).
2. Open the **Lead Webhook** node to copy the URLs:
   - **Test URL:** `http://localhost:5678/webhook-test/lead-capture` (works for one call after you click Execute workflow)
   - **Production URL:** `http://localhost:5678/webhook/lead-capture` (works only after you toggle the workflow **Active**)
3. In PowerShell, paste this helper once:

```powershell
function Send-Lead {
  param([string]$Json, [string]$Url = "http://localhost:5678/webhook-test/lead-capture", [string]$ContentType = "application/json", [hashtable]$Headers = @{})
  try {
    $r = Invoke-WebRequest -Method Post -Uri $Url -ContentType $ContentType -Headers $Headers -Body $Json -UseBasicParsing
    "$($r.StatusCode) $($r.Content)"
  } catch {
    if ($_.Exception.Response) { "$([int]$_.Exception.Response.StatusCode) $($_.ErrorDetails.Message)" } else { $_.Exception.Message }
  }
}
```

4. Send a lead:

```powershell
$lead = @{ name="Amit Sharma"; email="amit@example.com"; phone="9876543210"; company="ABC Solutions"; requirement="Website Development"; source="Website" } | ConvertTo-Json
Send-Lead -Json $lead
```

Expected: `200 {"success":true,"message":"Lead processed successfully","action":"created"}`, a new row in the sheet, and a Telegram message.

5. Click **Execute workflow** again (test URLs are single-use), then send the same payload: expected `action":"updated"`, no new row, Notes gains a "Repeat inquiry" line, Status unchanged.
6. Invalid input (also needs Execute workflow first):

```powershell
Send-Lead -Json '{"name":"","email":"not-an-email"}'
```

Expected: `400` with an `errors` list.

## 7. Activate workflow 01 and 02

1. In each workflow flip the **Inactive → Active** toggle (top right). Workflow 01 now answers on the **Production URL**.
2. Re-run the PowerShell tests with `-Url http://localhost:5678/webhook/lead-capture`.
3. Workflow 02 runs every hour at minute 0 (workflow timezone is `Asia/Kolkata`).

## 8. Test the follow-up reminder immediately (no waiting an hour)

1. Create a lead as in step 6 (Status `New`).
2. In the sheet, set that row's **Follow-up At** to a past time, e.g. `2020-01-01T10:00:00+05:30`, and make sure **Follow-up Reminder Sent** is empty.
3. Open workflow 02 and click **Execute workflow** on the **Run Now (Manual Test)** trigger (or use *Test workflow*).
4. Expected: one Telegram reminder, and **Follow-up Reminder Sent** now holds a timestamp.
5. Run it again: no second reminder (the row is skipped because Reminder Sent is filled).
6. To re-test, clear Reminder Sent.

**More reminders for untouched leads.** The default is one reminder per lead. To add an escalation (for example a second reminder after 24 more hours): add a column `Reminder 2 Sent` to the sheet; duplicate the `Filter Eligible Leads` → `Send Telegram Reminder` → `Update Reminder Fields` chain; in the copy require `Follow-up Reminder Sent` to be non-empty, `Reminder 2 Sent` empty, Status still `New`, and the time since `Follow-up Reminder Sent` over 24h; write the stamp to `Reminder 2 Sent`; and add that column to the schema list of the copied update node.

## 9. Error monitoring workflow

1. Open workflow 03, set `adminChatId` in Config, select the Telegram credential, save, and **activate** it.
2. In workflows 01 and 02: **⋯ → Settings → Error Workflow →** select *Error Monitoring* → Save.
3. The error workflow is triggered by failures of automatic (production) executions; failures during manual test runs may not trigger it, so test it by temporarily breaking something in an active workflow.
4. Note that workflow 01 and 02 already catch Google Sheets/Telegram failures themselves and send an admin alert, so workflow 03 mainly covers unexpected errors (for example a Code node error).
5. Messages contain the workflow name, failed node, mode, execution ID, and a redacted/truncated error message. No credentials or lead details are included.

## 10. Secure the webhook for production

Default is **no authentication** so you can test and so a plain browser form can post. Before exposing it publicly choose one:

- **Server-to-server callers (backend, Zapier-like tools, ad platforms that allow headers):** open **Lead Webhook** → *Authentication* → **Header Auth** → create credential: Name `X-Lead-Token`, Value = a long random string. Callers must send that header:
  ```powershell
  Send-Lead -Json $lead -Url http://localhost:5678/webhook/lead-capture -Headers @{ "X-Lead-Token" = "your-long-random-value" }
  ```
- **Public website form:** a secret placed in browser JavaScript is not secret. Put a small server-side proxy (your website backend, a serverless function, or a reverse-proxy rule) in front of n8n that adds the header, applies rate limiting and bot checks (CAPTCHA/honeypot), and forwards to n8n. Also set `corsAllowOrigin` to your site's exact origin.

## 11. Demo form and public access

1. Open `04_demo_lead_form.html`, set `WEBHOOK_URL` in the script, and open the file in a browser.
2. `http://localhost:5678/...` is reachable **only from your own computer**. Anyone else (visitors, Google Ads, WhatsApp/Meta) cannot reach it. For real traffic you need a public **HTTPS** URL: a reverse proxy with a domain and TLS certificate, a tunnel service (check its current free-tier terms), or a hosted n8n. When doing so set the `WEBHOOK_URL` environment variable on the n8n container to the public base URL so n8n displays correct webhook URLs.
3. The form sends the JSON as `text/plain` by default to avoid CORS preflight (n8n 2.15.1's Webhook node has no OPTIONS method/CORS setting, and I could not verify preflight behaviour). Workflow 01's validation parses both. If your browser reports a CORS error, check the Network tab; the response from n8n must include `Access-Control-Allow-Origin` (set by the Respond nodes from `corsAllowOrigin`). If the form still cannot post cross-origin, serve the page through the same origin or the proxy from step 10.

## 12. Logs and troubleshooting

- **Executions** tab (left sidebar or inside the workflow) → click a run → green/red nodes; click a node to see its input/output. Enable *Save successful executions* and *Save failed executions* in workflow Settings if runs are missing.
- Docker logs: `docker logs n8n --tail 100`.

| Symptom | Likely cause / fix |
|---|---|
| `404 The requested webhook is not registered` | Test URL used without clicking *Execute workflow*, or Production URL used while workflow is inactive |
| Sheets node: "Could not find sheet/ column" | Tab not named `Leads`, or `sheetName` in Config differs; headers misspelled |
| Sheets node: "The caller does not have permission" | The Google account in the credential cannot edit that spreadsheet |
| `Invalid submission` 400 for good-looking data | Check `errors` in the response; phone needs 7-15 digits |
| New lead saved but no Telegram message | Telegram credential/chat ID wrong; bot not started; the flow shows `Notification Failed (lead saved)` |
| Reminders never sent | Status not `New`, Follow-up At not ISO format, Reminder Sent not empty, or workflow 02 not active |
| Reminder sent twice | Telegram succeeded but the sheet update failed (Alert Admin fires); fix the sheet issue and clear the stamp manually |
| `Google hasn't verified this app` | Expected in Testing mode: Advanced → continue, and make sure your account is listed as a test user |

## 13. Backups and protecting credentials

- Export workflows regularly: in each workflow **⋯ → Download**, or CLI: `docker exec n8n n8n export:workflow --all --output=/home/node/.n8n/workflows-backup.json` then `docker cp n8n:/home/node/.n8n/workflows-backup.json .`
- Back up the Docker volume `n8n_data` (contains the database and the **encryption key** that protects stored credentials). Without that key, credentials cannot be decrypted after a restore. Store the key (`N8N_ENCRYPTION_KEY`, or the `encryptionKey` in `/home/node/.n8n/config`) in a password manager, separately from workflow backups.
- Never commit credentials, `.env` files, or decrypted credential exports to Git (see `.gitignore`). Workflow JSON from this project contains no credentials.
- Rotate the Telegram bot token with BotFather (`/revoke`) if it ever leaks; revoke Google access at https://myaccount.google.com/permissions.

## 14. Production deployment checklist

- [ ] n8n behind HTTPS (reverse proxy), `WEBHOOK_URL` set to the public URL, editor UI not open to the internet (restrict by VPN/IP/SSO).
- [ ] `N8N_ENCRYPTION_KEY` set explicitly and backed up.
- [ ] Header Auth or a server-side proxy in front of the webhook; rate limiting and bot protection; `corsAllowOrigin` set to your exact origin.
- [ ] Container restart policy (`--restart unless-stopped`) and a persistent volume or database; consider PostgreSQL for larger volumes.
- [ ] Execution pruning (`EXECUTIONS_DATA_PRUNE=true`, `EXECUTIONS_DATA_MAX_AGE`) so lead data is not stored indefinitely in execution logs.
- [ ] Error Workflow set on workflows 01 and 02; workflow 03 active.
- [ ] Google OAuth consent screen published (or accept 7-day token expiry) and the spreadsheet shared only with required people.
- [ ] Privacy notice on your form; a process for deleting lead data on request.
- [ ] Docker/n8n image updates scheduled and tested.
