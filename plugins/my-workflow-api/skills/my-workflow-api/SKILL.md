---
name: my-workflow-api
description: Webhook-triggered workflow automation for developers. Define trigger conditions and actions like send_email or slack_message, with variable resolution and automatic retries.
---

# MyWorkflowAPI

## Quick Start
1. Create a webhook endpoint via `my-webhook-api`, note its `id`.
2. `POST /workflow/workflows` with `org_id`, `trigger_config.endpoint_id`, and a `steps` array.
3. Enable it: `POST /workflow/workflows/{id}/enable`.
4. Any payload POSTed to the webhook URL will now trigger the workflow steps.

## Overview
Webhook-triggered workflow automation for developers. Define trigger conditions and multiple actions like `send_email` or `slack_message` in sequence, with variable resolution and automatic retries.

## Base URL
Production: `https://api.myworkflowapi.com`

## Authentication
Authentication is performed via an API key in the `Authorization` header.
Use `Bearer <api_key>` (the persistent key generated from `my-api-hq`).

## Core Endpoints

### 1. Workflows

**Create a Workflow:**
Create a workflow with a webhook trigger and a sequence of steps. Step configs support `{{payload.field}}` variable interpolation.
```http
POST /workflow/workflows
Authorization: Bearer <api_key>

{
  "org_id": "<uuid>",
  "name": "Send Welcome Email",
  "trigger_config": {
    "endpoint_id": "<webhook_endpoint_uuid>"
  },
  "steps": [
    {
      "type": "send_email",
      "from": "hello@example.com",
      "to": "{{payload.user.email}}",
      "subject": "Welcome, {{payload.user.first_name}}!",
      "template_id": "<email_template_uuid>"
    }
  ]
}
```

**List Workflows:**
```http
GET /workflow/workflows
Authorization: Bearer <api_key>
```

**Get a Workflow:**
```http
GET /workflow/workflows/{id}
Authorization: Bearer <api_key>
```

**Update a Workflow:**
Partially update a workflow. You can update `name`, `trigger_config`, or completely replace the `steps` array.
```http
PATCH /workflow/workflows/{id}
Authorization: Bearer <api_key>

{
  "name": "Updated Name",
  "steps": [
    {
      "type": "send_email",
      "from": "hello@example.com",
      "to": "different@example.com",
      "subject": "Updated Email",
      "template_id": "<email_template_uuid>"
    }
  ]
}
```

### Step Types

#### `send_email`
Sends an email. `template_id` takes priority over inline `html`.
**Prerequisite:** `from` must be a mailbox that has been created and sending-activated via `my-email-api`. Using an arbitrary address will silently fail.
```json
{
  "type": "send_email",
  "from": "hello@example.com",
  "to": "{{payload.email}}",
  "subject": "Hello {{payload.name}}",
  "template_id": "<email_template_uuid>"
}
```

**Important Note on Webhook Payloads:**
When a webhook receives a standard form submission (e.g. `email=test@test.com&name=Bob`), those fields are mapped directly onto the `payload` object. They are NOT wrapped in a `body` property. 
- **Correct:** `{{payload.email}}`
- **Incorrect:** `{{payload.body.email}}`

#### `slack_message`
Posts a message to a Slack channel via an incoming webhook URL. Supports `{{payload.*}}` variable interpolation in `text`.
```json
{
  "type": "slack_message",
  "webhook_url": "https://hooks.slack.com/services/T00/B00/xxx",
  "text": "New lead: {{payload.name}} from {{payload.company}} — {{payload.email}}"
}
```
- `webhook_url` — required. Create one at `api.slack.com/apps` → Incoming Webhooks.
- `text` — the message body. Supports `{{payload.field}}` and `{{payload.nested.field}}` interpolation.
- Failed sends are retried up to 3 times with exponential backoff (same as `send_email`).

**Enable a Workflow:**
```http
POST /workflow/workflows/{id}/enable
Authorization: Bearer <api_key>
```

**Disable a Workflow:**
```http
POST /workflow/workflows/{id}/disable
Authorization: Bearer <api_key>
```

**Delete a Workflow:**
```http
DELETE /workflow/workflows/{id}
Authorization: Bearer <api_key>
→ 204 No Content
```

### 2. Workflow Runs

**List Runs for a Workflow:**
Returns up to 100 most recent runs, ordered newest first.
```http
GET /workflow/workflows/{id}/runs
Authorization: Bearer <api_key>
→ [ { "id", "workflow_id", "trigger_payload", "status", "error", "attempt", "started_at", "finished_at", "created_at" } ]
```
`status` values: `pending`, `running`, `completed`, `failed`.

**Get a Single Run:**
```http
GET /workflow/runs/{id}
Authorization: Bearer <api_key>
→ { "id", "workflow_id", "trigger_payload", "status", "error", "attempt", "started_at", "finished_at" }
```
