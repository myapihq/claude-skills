---
name: my-pixel-api
description: >
  Query web visits, email events, and identity resolution data collected by the tracking pixel. Use this to retrieve engagement history, build attribution reports, or inspect the identity graph for a contact.
---

# MyPixelAPI Skill

## Quick Start
1. Run a campaign via `my-email-api` — each contact gets a unique `pixel_id` automatically.
2. `GET /pixel/interactions?campaign_id={id}` to see a unified timeline of opens, clicks, and visits.
3. `GET /pixel/identity/{pixel_id}` to resolve a contact's full identity graph.

Query interaction data collected by the tracking pixel — web visits, email events, and identity dossiers.

## Dependencies & Backlinks
- **Auth:** Requires a valid API key via `Bearer <api_key>` (from `my-api-hq`).
- **Data Source:** All interaction data is stored in the analytics backend.
- **Related:** Email campaign engagement stats are available via `my-email-api` (`/email/campaigns/{id}/stats`).

## Authentication
Use `Bearer <api_key>` in the `Authorization` header for all requests.

## Core Endpoints

> All list endpoints here support pagination via `?limit=` (default 50, max 500) and `?offset=`. Responses include `total`, `limit`, and `offset` fields.

### 1. Interactions (unified timeline)
Merges web visits and email events into a single timeline sorted by timestamp descending.
At least one of `website`, `domain`, or `campaign_id` is required.

```
GET https://api.mypixelapi.com/pixel/interactions
  ?website=yourdomain.com       # filter visits by destination domain (to_url)
  &domain=yourdomain.com        # filter email events by clicked URL domain (meta.url)
  &campaign_id=<id>             # filter email events by campaign
  &from=2024-01-01T00:00:00Z    # optional ISO8601 start
  &to=2024-12-31T23:59:59Z      # optional ISO8601 end
  &limit=50                     # default 50, max 500
  &offset=0

→ {
    "interactions": [
      { "type": "visit", "pixel_id": "...", "from_url": "...", "to_url": "...", "ts": "..." },
      { "type": "event", "pixel_id": "...", "event_type": "open|click|sent", "url": "...", "campaign_id": "...", "ts": "..." }
    ],
    "total_visits": N,
    "total_events": N,
    "limit": 50,
    "offset": 0
  }
```

### 2. Visits
Raw web visit log filtered by website domain.

```
GET https://api.mypixelapi.com/pixel/visits
  ?website=yourdomain.com
  &from=&to=&limit=&offset=

→ { "visits": [ { "pixel_id", "from_url", "to_url", "ts" } ], "total", "limit", "offset" }
```

### 3. Events
Raw email event log filtered by campaign or clicked URL domain.

```
GET https://api.mypixelapi.com/pixel/events
  ?campaign_id=<id>             # filter by campaign (utm_campaign)
  &domain=yourdomain.com        # filter by clicked URL domain (meta.url LIKE '%domain%')
  &from=&to=&limit=&offset=

→ { "events": [ { "pixel_id", "event_type", "meta", "ts" } ], "total", "limit", "offset" }
```
`meta` is a JSON string: `{"campaign_id": "...", "url": "..."}`. Note: `open` events always have `meta: "{}"` since the open pixel URL carries only `pid`.

### 4. Identity Dossier
Full BFS-resolved identity graph for a given pixel_id.

```
GET https://api.mypixelapi.com/pixel/identity/{pixel_id}

→ { "uuid": "...", "is_resolved": true, "nodes": { "<value>": { "type": "email|ip|...", "probability": 0.95 } }, "latency_ms": 12.3 }
```

## Event Types
| `event_type` | Meaning |
|---|---|
| `sent` | Email delivered to contact |
| `open` | Open pixel fired (email opened) |
| `click` | Link clicked in email |
| `page_visit` | Cookie bridge fired (landed on tracked page) |
