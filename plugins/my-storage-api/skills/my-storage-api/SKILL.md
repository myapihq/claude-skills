---
name: my-storage-api
description: >
  Upload and host static assets (images, files) and get back permanent public URLs. Use this to host images before embedding them in funnels or email templates.
---

# MyStorageAPI Skill

## Quick Start
1. `POST /hq/assets/ingest` with a public image URL, or `POST /hq/assets/upload` with a file.
2. Capture the `asset_id` and `url` from the response.
3. Pass the `url` into `my-funnel-api` pages or `my-email-api` templates.

Use this skill to store and list assets for web applications.

## Dependencies & Backlinks
- **Auth & Billing:** If you receive a 401 or 402, fall back to the `my-api-hq` skill to validate keys or top up.
- **Next Steps:** After storing assets, capture the returned URLs and pass them into the `asset_urls` parameter of the `my-funnel-api` when building pages.

## Authentication
Authentication is performed via an API key in the `Authorization` header.
Use `Bearer <api_key>` (the persistent key generated from `my-api-hq`).

## Core Workflows
1. **Ingest Asset**: 
   - **From URL:** `POST https://api.mystorageapi.com/hq/assets/ingest`
     - Payload: `{"url": "https://...", "name": "optional"}`
   - **From File (Multipart):** `POST https://api.mystorageapi.com/hq/assets/upload`
     - Payload: `multipart/form-data` with a `file` field containing the raw bytes. Optionally include `name` and `description` fields.
   - **From File (Raw Bytes):** `POST https://api.mystorageapi.com/hq/assets/upload`
     - Payload: Raw binary data in the body. Set the `Content-Type` to `image/jpeg` or `image/png`. You can optionally pass `?name=...&description=...` in the query string.
   - *Note: You can use `-k` with curl to bypass local dev SSL mismatches.*
   - **Format Rules:** ONLY `image/jpeg` and `image/png` are supported. WebP is rejected.
   - **Unsplash Pro-Tip:** If using Unsplash URLs, you MUST append `&fm=jpg` to force a JPEG response.
   - **Response:** `{ "asset_id": "img_...", "url": "https://api.myapihq.com/storage/img_...", "name": "...", "created_at": "..." }`
2. **List Assets**: `GET https://api.mystorageapi.com/hq/assets`
3. **Delete Asset**: `DELETE https://api.mystorageapi.com/hq/assets/{asset_id}`

## Payload Summary
| Endpoint | Field | Required | Description |
|---|---|---|---|
| `/ingest` | `url` | Yes | Publicly accessible URL of the asset (must be JPG or PNG). |
| `/ingest` | `name`, `description` | No | Optional labels for the asset. |
