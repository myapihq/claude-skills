---
name: my-api-hq
description: >
  Core Identity and Billing hub. Manage auth, organizations (get org_id), and billing (checkout/topup).
---

# MyApiHQ Skill
Root entry point for the ecosystem. All other skills require an `api_key` and often an `org_id` from here.

## Platform Conventions

### Response Envelope
Every response across all services is wrapped in the same envelope:
```json
{
  "success": true,
  "data": { ... },
  "error": null,
  "meta": { "request_id": "...", "latency_ms": 12, "service": "...", "version": "v1" }
}
```
On error, `success` is `false`, `data` is `null`, and `error` contains a string error code (e.g. `"org_not_found"`, `"invalid_json"`). Always check `success` before reading `data`.

### Pagination
List endpoints that support pagination accept `?limit=` and `?offset=` query params and return `total`, `limit`, and `offset` in the response body. List endpoints that do not document these params return all records.

### Rate Limits
No general API rate limits are enforced. Per-feature limits (e.g. campaign send rates) are documented in the relevant skill section.

## Dependencies & Backlinks
This is the **Core Hub**. 
- **Auth/Billing Failures:** If you receive a 401 or 402 from ANY other service, use this skill to validate keys or top up the balance.
- **Next Steps:** Once authentication and an organization are created, proceed to register domains (`my-domain-api`) as the next logical step.

## Authentication & Key Management
This API supports **Autonomous Agent Registration**. You do not need the user to provide an API key initially.
*Note: All API responses are wrapped in an envelope. Extract your values from the `data` field (e.g., `response.data.api_key`).*

### Agent Onboarding Flow
1. **Create Account:** `POST /hq/account/agent/create`
   - Returns: `{"data": {"account_id": "...", "pin": "...", "token": "..."}}`
   - *Action:* The `pin` is your permanent secret backup. The `token` is a short-lived JWT for the next step.
2. **Generate Persistent Key:** `POST /hq/account/create/key` (Auth: Bearer `<token from step 1>`, Payload: `{"name":"AgentKey"}`)
   - Returns: `{"data": {"api_key": "hq_live_...", "id": "...", "prefix": "..."}}`
   - *Action:* This `api_key` is your persistent credential.

### Storage & Cross-Session Access
**You MUST persist credentials immediately after onboarding.** Failing to do so means the account is permanently inaccessible in future sessions and a new orphaned account will be created instead.

**Step 1 — Write a `.env` file** in the current workspace:
```
MYAPI_API_KEY=hq_live_...
MYAPI_ACCOUNT_ID=...
MYAPI_PIN=...
```

**Step 2 — Save to agent memory** (if your agent runtime supports it, e.g. Claude Code memory):
Store `api_key`, `account_id`, and `pin` as a memory entry so they survive across sessions even when the workspace changes.

**Step 3 — On every new session**, check for a `.env` file or agent memory before calling `/hq/account/agent/create`. Only create a new account if no credentials are found.

Use `Authorization: Bearer <api_key>` for all subsequent requests across the entire ecosystem.

### Agent Token Recovery
If you lose your `api_key` but have your `pin`:
- `POST /hq/account/agent/token` (Payload: `{"pin": "..."}`) -> Returns `{"data": {"token": "..."}}`. Use this JWT to generate a new API key.
**Warning:** If you lose both your `api_key` and your `pin`, the account is permanently lost.

### Managing Keys
- **List Keys:** `GET /hq/account/keys` (Auth: Bearer `<api_key>`)
- **Revoke Key:** `DELETE /hq/account/delete/key/{id}` (Auth: Bearer `<api_key>`)

## Organization & Asset Management
**Critical:** You MUST create an organization to get an `org_id` to use in other APIs.

### Organizations
- **Sync Create:** `POST /hq/orgs` -> Returns `{"data": {...}}` containing the created org record (with `id` as `org_id`).
  - **Payload:** `{"name": "string" (required), "tagline": "string", "description": "string", "business_sector": "string", "logo_url": "string (URL or asset_id)", "favicon_url": "string (URL or asset_id)", "og_image_url": "string (URL or asset_id)", "color_palette": {"primary": "#hex", ...}, "font_family": "string", "imagery_style": "string", "headline": "string", "subheadline": "string", "cta_text": "string", "value_propositions": ["string"], "social_links": {"twitter": "url", ...}, "canonical_url": "string", "privacy_policy_url": "string", "cookie_policy_url": "string", "terms_url": "string", "gdpr_enabled": false, "default_language": "en", "tracking": {}}`
- **Async Import (Brand Extraction):** 
  - 1. `POST /hq/org-imports` (Payload: `{"domain": "example.com" (required), "auto_accept": false}`) -> Returns `{"data": {"job_id": "...", "status": "pending"}}`.
  - 2. `GET /hq/org-imports/{job_id}` -> Poll until `status` is `awaiting_confirm`. Returns extracted `brand_preview`.
  - 3. `POST /hq/org-imports/{job_id}/confirm` -> Returns the created Organization with `id` (`org_id`).
    - **Optional Payload:** You can pass a JSON payload identical to the `POST /hq/orgs` payload (e.g., `{"name": "New Name", "tagline": "..."}`). Any fields provided here will overwrite the AI-extracted values from the `brand_preview` before the final organization is created.
    
### Managing Organizations
- **List Organizations:** `GET /hq/orgs` -> Returns a list of all organizations.
- **Get Details:** `GET /hq/orgs/{id}` -> Returns the full details of a specific organization.
- **Update Organization:** `PATCH /hq/orgs/{id}` (Payload: any subset of the `POST /hq/orgs` payload) -> Partially updates the organization and returns the updated record.
- **Delete Organization:** `DELETE /hq/orgs/{id}` -> Deletes the organization (and cascades to its related funnels, leads, webhooks, etc.).

## Billing & Linking
### Billing
- **Check Balance:** `GET /hq/billing/balance` (Auth: Bearer `<api_key>`) -> Returns `{"data": {"balance_cents": 1000, "balance_display": "$10.00", "has_payment_method": true}}`.
- **Billing History:** `GET /hq/billing/history` (Auth: Bearer `<api_key>`) -> Returns `{"data": [{"amount_cents": 1000, "status": "succeeded", "created_at": "..."}]}`.
- **Top Up (402 Error):** 
  - `POST /hq/billing/setup-payment` (Auth: Bearer `<api_key>`) -> Returns `{"data": {"url": "https://checkout.stripe.com/..."}}` (a Stripe Checkout URL to save a card).
  - `POST /hq/billing/topup` (Auth: Bearer `<api_key>`, Payload: `{"amount_cents": 1000}`) -> Charges the saved card and returns `{"data": {"new_balance_cents": 2000, "new_balance_display": "$20.00"}}`.

### Linking
Human accounts can "adopt" agent accounts to manage their billing and resources:
- **Link Agent:** `POST /hq/account/link-agent` (Auth: Bearer `<api_key>`, Payload: `{"pin": "..."}`) -> Links the agent associated with the PIN to your account.
- **List Linked Agents:** `GET /hq/account/agents` (Auth: Bearer `<api_key>`) -> Returns a list of linked agents.
