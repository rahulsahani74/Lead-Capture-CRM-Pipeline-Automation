# Optional Integrations

None of these are required. The core system (Google Sheets + Telegram) imports and runs without any of them, and none of the delivered workflows reference them. This file lists what each integration involves so you can add it later. **Details change; confirm in the official docs before building.** I have not built or tested any of these, and I deliberately do not list OAuth scopes or endpoint details I cannot verify.

| Integration | n8n node (verified to exist in n8n-nodes-base 2.15.1) | What you need | Costs / approvals | Public HTTPS needed? |
|---|---|---|---|---|
| HubSpot CRM | `HubSpot` (n8n-nodes-base.hubspot) | HubSpot account; credential (private app token or OAuth2, per n8n's HubSpot credential docs) | Free CRM tier exists; some features/API limits depend on plan | No |
| Airtable | `Airtable` (n8n-nodes-base.airtable) | Airtable account, base, and a personal access token or OAuth2 credential | Free plan has record/API limits | No |
| Email inquiries | `Gmail Trigger` (n8n-nodes-base.gmailTrigger, polls Gmail via Google OAuth2) or `Email Trigger (IMAP)` (n8n-nodes-base.emailReadImap, any IMAP mailbox) | Google OAuth2 credential with Gmail API enabled, or IMAP host/user/app password | Free; Gmail needs Google Cloud setup like Sheets; IMAP may require an app password | No (polling) |
| WhatsApp Business Platform | `WhatsApp Business Cloud` (n8n-nodes-base.whatsApp) and `WhatsApp Trigger` (whatsAppTrigger) | Meta developer app, WhatsApp Business account, phone number, access token | Meta business verification and approval for production use; **per-conversation/message fees** apply beyond free allowances; message templates need approval | **Yes** (Meta calls your webhook) |
| Google Ads lead forms | Use a `Webhook` node (separate path, e.g. `google-ads-lead`) | Lead form asset configured in Google Ads with your webhook URL and a shared key | Google Ads account; ad spend | **Yes** (publicly reachable HTTPS URL) |
| Resend (email notifications) | No dedicated node verified; use `HTTP Request` (n8n-nodes-base.httpRequest) | Resend account, API key, verified sending domain | Free tier with limits; paid plans above that | No |

## How each would plug into this project

**General pattern.** All inbound sources should end up as the same JSON schema (`name`, `email`, `phone`, `company`, `requirement`, `source`) POSTed to the existing `lead-capture` webhook, or feed the same validation/matching nodes. That keeps duplicate detection in one place. Set `source` to the channel (e.g. `Email`, `WhatsApp`, `Google Ads`).

**Email inquiries.** Add a second workflow: Gmail Trigger (or IMAP Trigger) → Code node that extracts sender name/email and puts the subject/body into `requirement` → HTTP Request node POSTing to this project's own webhook Production URL. Be careful to filter out newsletters/auto-replies, or you will log them as leads.

**WhatsApp Business Platform.** Needs a Meta developer app, a registered business phone number and (for outbound messages outside the 24-hour window) approved templates. Inbound messages reach n8n only through a public HTTPS webhook URL that Meta can call, so local-only n8n will not work. WhatsApp numbers typically arrive in international format without a `+`; the duplicate matching compares digits only, so such numbers will not match leads stored in national format (e.g. `9876543210`).

**Google Ads lead forms.** Google can deliver form submissions to a webhook URL you provide; it requires a publicly reachable HTTPS endpoint and has a way to send a verification key you should check in a `Code`/`IF` node before processing. Check Google's current documentation for the exact payload field names (they differ from this project's schema, so add a mapping Code node) and for how to test the webhook.

**HubSpot / Airtable.** Add a node after `Append New Lead` / `Update Existing Lead` that creates or updates the record. Treat Google Sheets as the source of truth until you are confident; give the extra node `Continue (using error output)` so a CRM failure does not break the response to the visitor. Use email as the unique key in both systems.

**Resend.** Create an `HTTP Request` node (POST, JSON body, `Authorization: Bearer <key>` through an n8n *Header Auth* credential, never pasted into the node). Use it for an email copy of the new-lead notification or an auto-reply to the lead. Auto-replies to the lead require a verified domain and should respect consent/anti-spam laws.

## What you must not do

- Do not hardcode API keys in nodes, in the demo HTML, or in committed files; use n8n credentials.
- Do not expose the n8n editor to the internet to receive these webhooks; expose only the `/webhook/` path through a reverse proxy.
