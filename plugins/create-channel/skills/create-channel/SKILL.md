---
name: create-channel
description: Create a FinalApproval channel to add human-in-the-loop approval to your AI agent workflows
argument-hint: <description of what needs approval, e.g. "email sending", "deployments", "billing charges">
---

# Create FinalApproval Channel

Set up a human-in-the-loop approval channel so AI agent actions require human review before executing.

## Steps

### 1. Understand the use case

Ask the developer what action needs human approval. Common examples:
- **Send email** — recipient, subject, body
- **Deploy to production** — service, environment, commit hash, diff link
- **Charge customer** — customer name, amount, description
- **Delete resource** — resource type, resource name, reason

### 2. Design the UI schema

Based on the use case, define an ordered array of fields. Each field has:

```json
{
  "key": "field_name",
  "label": "Human-readable Label",
  "type": "text | code | link | badge | longtext"
}
```

**Type reference:**
- `text` — short string (names, emails, IDs)
- `code` — monospace block (code snippets, diffs, JSON)
- `link` — clickable URL
- `badge` — small tag/label (status, priority, environment)
- `longtext` — multi-line text with preserved whitespace (email body, descriptions)

**Example for "send email":**
```json
[
  { "key": "recipient", "label": "To", "type": "text" },
  { "key": "subject", "label": "Subject", "type": "text" },
  { "key": "body", "label": "Body", "type": "longtext" }
]
```

### 3. Create the channel via API

The developer must be logged in and have their session cookie. Use the dashboard UI at `http://localhost:5173/dashboard/channels/new` or call the API directly:

```bash
curl -X POST http://localhost:3001/api/channels \
  -H "Content-Type: application/json" \
  -H "Cookie: <session_cookie>" \
  -d '{
    "name": "Send Email",
    "description": "Approve outgoing emails before sending",
    "ui_schema": [
      { "key": "recipient", "label": "To", "type": "text" },
      { "key": "subject", "label": "Subject", "type": "text" },
      { "key": "body", "label": "Body", "type": "longtext" }
    ]
  }'
```

The response includes the channel ID and a one-time **API key** (prefixed with `fa_`). Save this key immediately — it cannot be retrieved again.

### 4. Save the API key

Add the API key to the developer's `.env` file:

```
FINALAPPROVAL_API_KEY=fa_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
FINALAPPROVAL_CHANNEL_ID=<channel-uuid>
```

### 5. Add the approval call to the codebase

At the point in the code where the action should be gated by human approval, add a POST request:

```typescript
const response = await fetch("http://localhost:3001/api/v1/approvals", {
  method: "POST",
  headers: {
    "Authorization": `Bearer ${process.env.FINALAPPROVAL_API_KEY}`,
    "Content-Type": "application/json",
  },
  body: JSON.stringify({
    title: "Send email to user@example.com",
    data: {
      recipient: "user@example.com",
      subject: "Welcome to our platform",
      body: "Hello! Thanks for signing up..."
    }
  }),
});

const { id, status } = await response.json();
// status will be "pending" — the human must approve in the dashboard
```

### 6. Confirm setup

Tell the developer:
1. The channel is live at their FinalApproval dashboard
2. Approval requests will appear there when the AI agent sends them
3. They can approve or deny each request from the dashboard
4. Webhook support for async approve/deny callbacks is coming soon
