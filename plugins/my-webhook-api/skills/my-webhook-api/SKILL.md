---
name: my-webhook-api
description: Inbound webhook infrastructure for developers. Create unique endpoints, receive payloads, and fan out to workflow automations in real time.
---

# MyWebhookAPI

## Quick Start
1. `POST /webhook/endpoints` with `org_id` and a name — get back an `id` and inbound `slug`.
2. Give the inbound URL (`/webhook/in/{slug}`) to a third party or use it as an HTML form `action`.
3. Wire the endpoint to a workflow via `my-workflow-api` to act on incoming payloads.

## Overview
Inbound webhook infrastructure for developers. Create unique endpoints, receive payloads, and fan out to workflow automations in real time.

## Base URL
Production: `https://api.mywebhookapi.com`

## Authentication
Authentication is performed via an API key in the `Authorization` header.
Use `Bearer <api_key>` (the persistent key generated from `my-api-hq`).

## Core Endpoints

### 1. Webhook Endpoints (Provisioning)

**Create an Endpoint:**
Provisions a new inbound endpoint with a unique URL. Payloads POSTed to the URL are stored and fanned out to connected workflows.
```http
POST /webhook/endpoints
Authorization: Bearer <api_key>

{
  "org_id": "<uuid>",
  "name": "Checkout webhooks",
  "description": "Optional description"
}
```

**List Endpoints:**
```http
GET /webhook/endpoints
Authorization: Bearer <api_key>
```

**Delete an Endpoint:**
```http
DELETE /webhook/endpoints/{id}
Authorization: Bearer <api_key>
```

### 2. Receiving Payloads

**Inbound Webhook URL:**
This is the URL you give to third parties (like Stripe, GitHub, etc.) or use in native HTML `<form method="POST">` actions.
```http
POST /webhook/in/{slug}
```
**Supported Content-Types:**
- `application/json`: Standard JSON payload.
- `application/x-www-form-urlencoded`: Standard HTML form submission. The key-value pairs are automatically converted to a JSON object for use in workflows.
- `multipart/form-data`: HTML form submission with files or advanced encoding. The key-value fields are automatically parsed and converted to a JSON object.

**JSON Example:**
```json
{
  "any": "JSON payload"
}
```

**Form Example:**
```html
<form method="POST" action="https://api.mywebhookapi.com/webhook/in/your-slug">
  <input name="email" value="user@example.com">
  <input name="first_name" value="Riccardo">
  <button type="submit">Subscribe</button>
</form>
```
When this form is submitted (or equivalent JSON POSTed), the internal payload object available to workflows will be:
```json
{
  "email": "user@example.com",
  "first_name": "Riccardo"
}
```
This means a connected workflow can use `{{payload.email}}` and `{{payload.first_name}}`. Note that there is no `.body` or `.data` wrapper — the submitted fields become the direct properties of the `payload` object.

### 3. Deliveries

**Get a Delivery Record:**
View the status, payload, and headers of a specific webhook delivery.
```http
GET /webhook/deliveries/{id}
Authorization: Bearer <api_key>
```
