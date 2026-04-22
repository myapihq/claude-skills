---
name: my-image-api
description: >
  AI image generation with automatic cloud storage. Pass a prompt, get a
  hosted public URL. Async job-based workflow powered by Gemini.
---

# MyImageAPI Skill

## Quick Start
1. `POST /image/generate` with a `prompt`.
2. Poll `GET /image/jobs/{job_id}` every 3–5 seconds until `status` is `completed` (give up after ~60 seconds / 15 attempts).
3. Use the returned `url` directly in funnels or email templates.

Generate images from natural language prompts. Every image is automatically
stored and returned as a permanent public URL — no upload step required.

## Dependencies & Backlinks
- **Auth & Billing:** Fall back to the `my-api-hq` skill if you receive a 401 or 402.
- **Next Steps:** Capture the returned `url` and pass it into `my-funnel-api` (as `asset_urls`) or `my-email-api` templates to embed the image.

## Authentication
Authentication is performed via an API key in the `Authorization` header.
Use `Bearer <api_key>` (the persistent key generated from `my-api-hq`).

## Pricing
**$0.05 per generated image** (5 credits). Charged at job creation. Refunded automatically if generation fails.

## Core Workflow

### 1. Generate an image (async)
```
POST https://api.myimageapi.com/image/generate
{
  "prompt": "minimalist hero background, dark blue gradient, no text",
  "aspect_ratio": "16:9",
  "style": "photorealistic",
  "colors": "#1a1a2e, #e94560",
  "has_text": false
}
→ { "job_id": "a3f1c2d4-...", "status": "pending" }
```

### 2. Poll until completed
```
GET https://api.myimageapi.com/image/jobs/{job_id}
→ { "job_id": "...", "status": "completed", "url": "https://api.myapihq.com/storage/img_...", "prompt": "...", "aspect_ratio": "16:9", "created_at": "..." }
```
Poll every 3–5 seconds until `status` is `completed` or `failed`.

### 3. List all generated images
```
GET https://api.myimageapi.com/image/list
→ [ { "job_id", "status", "url", "prompt", "aspect_ratio", "created_at" } ]
```

## Payload Summary
| Endpoint | Field | Required | Description |
|---|---|---|---|
| `/image/generate` | `prompt` | Yes | Natural language description of the image. |
| `/image/generate` | `aspect_ratio` | No | `1:1` (default), `16:9`, `9:16`, `4:3`, `3:4`. |
| `/image/generate` | `style` | No | Style hint e.g. `photorealistic`, `flat illustration`, `watercolor`. |
| `/image/generate` | `colors` | No | Color palette hint e.g. `#1a1a2e, #e94560`. |
| `/image/generate` | `has_text` | No | Allow text/logos in the image (default `false`). |

## Job Statuses
| Status | Meaning |
|---|---|
| `pending` | Queued, not yet started. |
| `processing` | Gemini generation in progress. |
| `completed` | Done — `url` field contains the public image link. |
| `failed` | Generation failed — `error` field has the reason. Credits refunded. |
