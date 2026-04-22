---
name: my-email-api
description: >
  Email infrastructure. Mailbox creation, sending, reading, email verification,
  and AI-powered email template generation with open/click/page tracking.
---

# MyEmailAPI Skill

## Quick Start
1. Register a domain via `my-domain-api`, then `POST /email/mailbox/create`.
2. `POST /email/sending/activate` to unlock outbound sending for that address.
3. Generate a template (`POST /email/templates/generate`), poll until `completed`, then `POST /email/send`.

Transactional email engine with AI template builder and tracking.

## Dependencies & Backlinks
- **Auth & Billing:** Fall back to the `my-api-hq` skill if you receive a 401 or 402.
- **Domain Prerequisite:** You MUST have an owned domain before creating a mailbox. If you do not have one, register it via `my-domain-api` first.
- **Org Prerequisite:** Templates are org-scoped. You need an `org_id` from `my-api-hq` before generating templates.
- **Next Steps:** If setting up an email for lead capture, use `my-funnel-api` to build the landing page.

## Authentication
Authentication is performed via an API key in the `Authorization` header.
Use `Bearer <api_key>` (the persistent key generated from `my-api-hq`).

## Core Workflows

### 1. Mailbox & Sending
1. **Create mailbox**: `POST https://api.myemailapi.com/email/mailbox/create` ($0.50). Returns `address`.
2. **List mailboxes**: `GET https://api.myemailapi.com/email/mailbox/list/me` → array of mailboxes.
3. **Activate sending**: `POST https://api.myemailapi.com/email/sending/activate` ($2.50).
4. **Send**: `POST https://api.myemailapi.com/email/send`.
5. **Monitor**: 
   - `GET https://api.myemailapi.com/email/status/{id}` (sent message status)
   - `GET https://api.myemailapi.com/email/inbox/{address}` (received messages)
   - `GET https://api.myemailapi.com/email/message/{message_id}?address={address}` (read full content of a received message)

### 2. Email Warmup
Automatically warm up your email addresses to improve sender reputation and deliverability.
Requires sending to be activated first.

1. **Start Warmup**: `POST https://api.myemailapi.com/email/warmup/start`
   ```json
   { "address": "hello@example.com" }
   ```
   Deducts a one-time fee of $5.00 ($500 cents) and automatically configures the IMAP/SMTP account in Instantly.ai. Returns `warmup_status: "active"`.
2. **Pause Warmup**: `POST https://api.myemailapi.com/email/warmup/pause`
3. **Resume Warmup**: `POST https://api.myemailapi.com/email/warmup/resume`
4. **Stop Warmup**: `POST https://api.myemailapi.com/email/warmup/stop`
5. **Warmup Stats**: `GET https://api.myemailapi.com/email/warmup/stats/{address}`
   Returns Instantly analytics (e.g., `sent`, `landed_inbox`, `landed_spam`, `health_score`).

### 3. Email Templates (AI-powered)
Generate, edit, preview, and send HTML email templates using Gemini. Templates are org-scoped and inherit brand config (colors, logo, font).

**Generate from prompt (async):**
```
POST https://api.myemailapi.com/email/templates/generate
{ "org_id": "<org_id>", "prompt": "Welcome email for SaaS trial users, dark bg, CTA to dashboard", "name": "Welcome — Trial" }
→ { "job_id": "...", "template_id": "...", "status": "processing", "preview_url": "..." }
```
Poll job status until `status` is `completed` or `failed` (give up after ~90 seconds / 18 attempts at 5s intervals):
```
GET https://api.myemailapi.com/email/template-jobs/{job_id}
→ { "status": "processing"|"completed"|"failed", "template_id": "...", "result": { "template_id", "subject", "preview_url" } }
```
The `preview_url` is public — open directly in a browser, no auth needed. It auto-refreshes while the template is still generating.

**Edit via prompt (patch):**
```
POST https://api.myemailapi.com/email/templates/{id}/edit
{ "prompt": "Make the CTA button red and add an unsubscribe link in the footer" }
→ { "template_id": "...", "preview_url": "..." }
```

**Preview (Gmail-skin, browser):**
```
GET https://api.myemailapi.com/email/templates/{id}/preview?viewport=desktop|mobile
→ HTML page with Gmail chrome, desktop/mobile toggle
```

**Send test to inbox:**
```
POST https://api.myemailapi.com/email/templates/{id}/send-test
{ "to": "you@example.com" }
→ { "ok": true, "message_id": "..." }
```

**List templates:**
```
GET https://api.myemailapi.com/email/templates
→ [ { "id", "name", "subject", "preview_url", "created_at", "updated_at" } ]
```

**Delete:**
```
DELETE https://api.myemailapi.com/email/templates/{id}
→ 204 No Content
```

### 3. Tracking (open, click, page visit)
Tracking is injected automatically when sending via a template. Links are wrapped with click redirectors and a 1×1 open pixel is appended. All events are logged to an identity graph for real-time attribution.

- **Open pixel:** `GET /t/o?lid=<lead_id>` — Logs an `open` event.
- **Click redirect:** `GET /t/c?lid=<lead_id>&url=<dest>` — Logs a `click` event, then 302-redirects to the destination with the `lid` appended.
- **Cookie bridge:** `GET /t/p?lid=<lead_id>&page=<path>` — Sets the `_hspx` cookie on the target domain, stitching the email identity (`lead_id`) to the visitor's pixel graph.

**Querying tracking data:** All logged events are queryable via the `my-pixel-api` skill. Use `GET /pixel/interactions?campaign_id={id}` for a unified timeline, or `GET /pixel/identity/{pixel_id}` to resolve a contact's full identity graph.

**Identity Graph Matching:**
When a user clicks from an email (`lead_id`) to a website, the system links their historical anonymous behavior (tracked via `_hspx` cookie) to their email identity. This allows for deep attribution: "This lead opened the email, browsed the pricing page 3 times, and then submitted a form."

### 4. Email Campaigns
Send bulk email campaigns with rate limiting, async contact validation, and pixel-based engagement tracking.

**Rate limits (enforced, not configurable above):** max 3 emails/min, max 100 emails/hour per campaign.

**Full workflow:**

**1. Create campaign:**
```
POST https://api.myemailapi.com/email/campaigns
{ "name": "My Campaign", "template_id": "<template_id>", "from_address": "hello@example.com",
  "rate_per_minute": 3, "per_hour_limit": 100 }
→ { "id": "...", "status": "draft", ... }
```

**2. Upload contacts (validation starts automatically):**

Option A — JSON array:
```
POST https://api.myemailapi.com/email/campaigns/{id}/contacts/upload-list
{ "emails": ["a@example.com", "b@example.com"] }
→ { "list_id": "...", "total": 2, "status": "validating" }
```

Option B — CSV/TXT file upload (`multipart/form-data`, field name `file`):
```
POST https://api.myemailapi.com/email/campaigns/{id}/contacts/upload-file
Content-Type: multipart/form-data; boundary=...
(file field: CSV or plain-text file)
→ { "list_id": "...", "total": N, "status": "validating" }
```
CSV auto-detection: if a column header named `email` (case-insensitive) is present it is used; otherwise the first column is assumed. Duplicates are removed automatically.

Validation checks: email format, domain structure, MX record existence. Only valid contacts will be sent to.

**3. Poll validation status:**
```
GET https://api.myemailapi.com/email/campaigns/{id}/contacts/status
→ { "upload_status": "ready"|"validating"|"failed", "total": N, "valid_count": N, "invalid_count": N }
```
Wait until `upload_status` is `ready` before starting.

**4. Start sending:**
```
POST https://api.myemailapi.com/email/campaigns/{id}/start
→ { "status": "active" }
```
Errors if still validating or no valid contacts exist.

**5. Pause / Resume:**
```
POST https://api.myemailapi.com/email/campaigns/{id}/pause
POST https://api.myemailapi.com/email/campaigns/{id}/resume
```

**6. Stats (includes engagement from pixel tracking):**
```
GET https://api.myemailapi.com/email/campaigns/{id}/stats
→ {
    "campaign": { "id", "name", "status", "rate_per_minute", "per_hour_limit", ... },
    "validation": { "total": N, "valid": N, "invalid": N, "pending": N, "list_status": "ready" },
    "dispatch": { "sent": N, "remaining": N, "sent_this_hour": N, "hour_limit": 100, "last_dispatch_at": "..." },
    "engagement": { "sent": N, "opens": N, "clicks": N, "page_visits": N }
  }
```
Engagement stats (opens, clicks, page_visits) are powered by the pixel graph — each contact receives a unique `pixel_id` at send time, automatically tracked via the existing `/t/o` and `/t/c` infrastructure.

**List/Get/Update campaigns:**
```
GET https://api.myemailapi.com/email/campaigns
GET https://api.myemailapi.com/email/campaigns/{id}
PATCH https://api.myemailapi.com/email/campaigns/{id} (Payload: {"name": "...", "per_day_limit": 250})
```

**Template placeholder:** Only `{{domain}}` is supported (replaced with the contact's email domain). No other variable substitution.

## Payload Summary
| Endpoint | Field | Required | Description |
|---|---|---|---|
| `/mailbox/create` | `domain`, `username` | Yes | Organization-owned domain and desired prefix. |
| `/sending/activate` | `address` | Yes | The full email address to unlock. |
| `/send` | `from`, `to`, `subject` | Yes | `to` is an array of recipients. |
| `/send` | `html`, `text` | Yes | At least one body type must be provided. |
| `/templates/generate` | `org_id`, `prompt` | Yes | Natural language description of the email. |
| `/templates/{id}/edit` | `prompt` | Yes | Natural language change to apply. |
| `/templates/{id}/send-test` | `to` | Yes | Recipient for test send (no tracking injected). |
| `/campaigns/{id}/contacts/upload-list` | `emails` | Yes | JSON array of email addresses. |
| `/campaigns/{id}/contacts/upload-file` | `file` | Yes | CSV or plain-text file (multipart field `file`). |
