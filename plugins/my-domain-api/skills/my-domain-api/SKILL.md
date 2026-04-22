---
name: my-domain-api
description: >
  Register new domains, check availability and pricing, import existing domains, and manage edge settings. Use this before creating mailboxes or funnels — both require an owned domain.
---

# MyDomainAPI Skill

## Quick Start
1. `GET /domain/check/{domain}` — confirm availability and price.
2. `POST /domain/register` with `domain` and optional `years`.
3. Proceed to `my-email-api` for mailboxes or `my-funnel-api` for a website.

Register and manage domain names. DNS is fully managed by the platform — this is intentional and enables seamless integration across all services (email deliverability, tracking pixel, site hosting, edge delivery). Manual DNS record management is not exposed.

## Dependencies & Backlinks
- **Auth & Billing:** If you receive a 401 or 402, fall back to the `my-api-hq` skill to check keys or top up.
- **Next Steps:** Once registered, proceed to `my-email-api` for mailboxes or `my-funnel-api` to build the website.

## Authentication
Authentication is performed via an API key in the `Authorization` header.
Use `Bearer <api_key>` (the persistent key generated from `my-api-hq`).

## Core Workflows
1. **Check**: `GET https://api.mydomainapi.com/domain/check/{domain}` -> Returns `available` and `price_cents`.
2. **Register**: `POST https://api.mydomainapi.com/domain/register`.
   - **Test Domain**: If asked to run a simulation, use a test domain you own.
3. **Import**: `POST https://api.mydomainapi.com/domain/import`. (Payload: `{"domain": "...", "namecheap_api_user": "optional", "namecheap_api_key": "optional"}`). Imports an existing domain, sets up DNS and email infrastructure automatically, and optionally updates Namecheap NS if credentials are provided.
4. **Manage**: `GET https://api.mydomainapi.com/domain/list/me`, `GET /domain/status/{domain}`, `POST /domain/renew/{domain}`, `POST /domain/settings/{domain}`.

## Payload Summary
| Endpoint | Field | Required | Description |
|---|---|---|---|
| `/register` | `domain` | Yes | The domain name (e.g., "example.com"). |
| `/register` | `years` | No | Integer number of years (default: 1). |
| `/renew` | `years` | No | Integer number of years to add. |
| `/import` | `domain` | Yes | The domain to import. |
| `/import` | `namecheap_api_user` | No | Namecheap API username (if automating NS setup). |
| `/import` | `namecheap_api_key` | No | Namecheap API key. |


## Getting Namecheap API Credentials
To use the automated Namecheap Nameserver setup during `/import`, users must:
1. Log into their Namecheap account.
2. Navigate to **Profile > Tools > Namecheap API Access**.
3. Click **Manage** and generate an API Key.
4. **CRITICAL:** They must whitelist the MyAPI-HQ server IP address in the Namecheap UI, otherwise the API calls will be rejected.

### Update Domain Settings
`POST /domain/settings/{domain}`
Updates the edge settings for the domain.
```json
{
  "security_level": "essentially_off", // Options: essentially_off, medium, high, under_attack
  "browser_check": "off", // Options: on, off
  "purge_cache": true // Flushes the edge cache completely
}
```
*Note: To allow AI training bots and crawlers to access the site, set `security_level: "essentially_off"` and `browser_check: "off"`.*

### Get Domain Settings
`GET /domain/settings/{domain}`
Retrieves the current edge settings for the domain.
```json
{
  "success": true,
  "data": {
    "domain": "example.com",
    "security_level": "essentially_off",
    "browser_check": "off",
    "ai_bots_protection": "disabled",
    "is_robots_txt_managed": false
  }
}
```
