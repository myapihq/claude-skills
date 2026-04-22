---
name: my-funnel-api
description: >
  Host multi-page sites and funnels by pushing your own raw HTML to named slugs. Handles routing, edge delivery, webhook wiring, and link/workflow verification. You write the HTML; the platform hosts it.
---

# MyFunnelAPI Skill

## Quick Start
1. Create an org (`my-api-hq`) and register a domain (`my-domain-api`).
2. `POST /funnel/funnels/create-raw` with `org_id` and `domain`.
3. `POST /funnel/funnels/{id}/push-page` with `slug: "/"` and your HTML.

Headless site hosting. You write the HTML; this skill handles the infrastructure (routing, edge delivery, webhook listener registration). There is no AI HTML generation or editing — the platform only hosts and serves what you push.

## Dependencies & Backlinks
- **Auth & Billing:** Fall back to the `my-api-hq` skill if you receive a 401 or 402.
- **Organization Prerequisite:** You MUST create an Organization in `my-api-hq` first.
- **Domain Prerequisite:** The domain MUST be registered via `my-domain-api` before you can create a funnel on it.

## Authentication
Use `Bearer <api_key>` in the `Authorization` header.

## Core Workflow (Headless)

### Step 1 — Create Infrastructure
`POST /funnel/funnels/create-raw`
Send `org_id` and `domain`.
This creates the funnel record in the database, auto-registers the organization webhook listener, and provisions the edge routing for the domain.
Returns the funnel object including its `id`.

### Step 2 — Verify HTML (Optional but recommended)
`POST /funnel/funnels/{id}/verify`
Send EITHER `html` (to test locally) OR `slug` (to fetch the live HTML from KV).

This acts as a pre-launch linter. It runs three categories of checks concurrently:

**1. Links and images** — parses every `<a href>` and `<img src>`, fires concurrent network requests (Range GET with HEAD fallback). Internal root-relative paths (e.g. `/about`) are checked against the published KV store for that funnel instead.

**2. Webhook + workflow integrity** — extracts every `/webhook/in/{slug}` reference from the HTML and for each one:
- Checks the webhook endpoint exists in the database
- Checks at least one enabled workflow is bound to it
- Validates each workflow's steps:
  - `send_email`: must have `from`, `to`; if `template_id` is set, the template must exist
  - `slack_message`: must have `webhook_url`

**3. Modes and response shape**

| Body | Behaviour | Response key |
|------|-----------|--------------|
| `{}` or no body | Verify **all** published pages (default) | `pages` (array) |
| `{"slug": "/about"}` | Verify one specific slug | `tests` (array) |
| `{"html": "..."}` | Verify raw HTML before publishing | `tests` (array) |

All-pages response:
```json
{
  "pages": [
    {
      "slug": "/",
      "tests": [
        { "type": "link",    "url": "https://...",          "passed": true,  "details": "HTTP 200 OK" },
        { "type": "webhook", "url": ".../webhook/in/abc123", "passed": true,  "details": "2 workflow(s) bound and valid: \"Lead Capture\"" }
      ]
    },
    {
      "slug": "/blog/post-slug",
      "tests": [
        { "type": "webhook", "url": ".../webhook/in/def456", "passed": false, "details": "Webhook endpoint exists but has no enabled workflow — form submissions will be silently dropped" }
      ]
    }
  ]
}
```

### Step 3 — Push Page HTML
`POST /funnel/funnels/{id}/push-page`
Send `slug` (e.g., `"/"`) and `html` (the raw HTML string).
This directly overwrites the published HTML for that slug. The site is immediately live.
Returns `{ "status": "pushed", "key": "..." }`.

## Manage funnels
- `GET /funnel/funnels` — list all funnels.
- `GET /funnel/funnels/{id}` — get funnel metadata.
- `DELETE /funnel/funnels/{id}` — permanently delete funnel and purge all KV snapshots.

## Payload Summary
| Endpoint | Required Fields | Description |
|---|---|---|
| `/create-raw` | `org_id`, `domain` | Setup database and edge infrastructure. |
| `/verify` | `html` OR `slug` | Lint links, images, and webhook→workflow integrity. |
| `/push-page` | `slug`, `html` | Upload raw HTML to the live site. |

## Notes for AI Agents
- This is the canonical site/funnel builder. There is no separate `my-website-api`.
- **You generate the HTML.** Write complete, self-contained HTML pages and push them via `push-page`. The platform does not generate or modify HTML for you.
- `slug` must always start with a forward slash (e.g., `/`).
- Assets (images, CSS) should be hosted via `my-storage-api` and referenced by their returned `url`.
