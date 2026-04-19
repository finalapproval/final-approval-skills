---
name: create-channel
description: Create a FinalApproval channel to add human-in-the-loop approval to your AI agent workflows
argument-hint: <description of what needs approval, e.g. "email sending", "deployments", "billing charges">
---

# Create FinalApproval Channel

Wire human-in-the-loop approval into an agent's action. The end result: the agent's code submits a request, a human approves or denies in the dashboard, and the webhook fires back into the agent's code to execute or abort.

## Steps

### 1. Scope the channel — minimal questions, smart defaults

Only ask what actually changes the code. Everything else, infer or pick a sensible default and move on.

**Q1 — Pick the action type** (single multiple-choice question):

> "What kind of action are we gating? Pick one:
> **a)** Send email / messaging
> **b)** Deploy / infra change
> **c)** Charge / refund / financial
> **d)** Post to social / publish content
> **e)** Database migration / destructive query
> **f)** Other (describe in one line)"

Each choice maps to a preset: channel name, description, TypeScript interface, and HTML card template. The presets are in the appendix. **Use them as-is** unless the user picks (f) or pushes back.

**Q2 — Confirm data fields** (only if needed):

Show the preset's data fields and ask one question:

> "I'll send these fields with each approval: `recipient`, `subject`, `body`, `priority`. **Add or remove any?** (reply 'looks good' to accept)"

If they accept, move on. If they edit, update the interface and HTML template — don't re-ask anything else.

**Q3 — Webhook destination** (multiple choice):

> "Where will FinalApproval POST the approval decision? Pick one:
> **a)** I have a public HTTPS URL ready (paste it)
> **b)** Local dev — set up an ngrok / cloudflared tunnel
> **c)** No web server — scaffold a tiny Express receiver in this project
> **d)** Serverless (Vercel / Cloudflare Workers / Lambda)"

Their choice drives step 7's scaffolding. No follow-up questions — pick a default port (3000) and route (`/webhooks/finalapproval`) and run with it.

**Don't ask about:**
- *Why* approval is needed — doesn't change the code
- Reviewer persona — the preset templates already render well for both technical and non-technical reviewers
- Frequency / urgency — doesn't affect channel creation; notification settings live in the dashboard
- Trigger location — `Grep` the codebase for the action's function call (e.g. `sendEmail(`, `deployService(`, `stripe.charges.create`) and surface candidates. Only ask if grep returns zero or many ambiguous matches.

**Don't summarize back** unless something is genuinely ambiguous. Move straight to step 2.

### 2. Authenticate (device flow)

The rest of this skill needs a bearer token. Tokens are stored at `~/.finalapproval/token.json`.

The default host is `https://www.finalapproval.ai`. Override with `FINALAPPROVAL_URL` (e.g. `http://localhost:3001` for local dev).

**2a. Check for an existing session — and offer to switch accounts:**

```bash
FINALAPPROVAL_URL="${FINALAPPROVAL_URL:-https://www.finalapproval.ai}"

CURRENT_EMAIL=""
if [ -f ~/.finalapproval/token.json ]; then
  TOKEN=$(jq -r .token ~/.finalapproval/token.json)
  # Verify it still works and get the signed-in user
  SESSION=$(curl -s -H "Authorization: Bearer $TOKEN" \
    -H "Origin: $FINALAPPROVAL_URL" \
    "$FINALAPPROVAL_URL/api/auth/get-session")
  CURRENT_EMAIL=$(echo "$SESSION" | jq -r '.user.email // empty')
  [ -z "$CURRENT_EMAIL" ] && rm ~/.finalapproval/token.json
fi
```

If a valid session exists, **always ask** the developer whether to continue as `$CURRENT_EMAIL` or switch accounts. Do not silently reuse the session.

- If they want to continue → skip to step 3.
- If they want to switch → `rm ~/.finalapproval/token.json` and fall through to 2b. The device flow starts fresh, no extra confirmation.

If no valid session exists, continue to 2b directly.

**2b. Request a device code and open the browser:**

Better Auth exposes an OAuth 2.0 device-authorization flow under `/api/auth/device/*`.

```bash
ORIGIN="$FINALAPPROVAL_URL"  # Better Auth requires an Origin header

RESP=$(curl -s -X POST "$FINALAPPROVAL_URL/api/auth/device/code" \
  -H "Content-Type: application/json" \
  -H "Origin: $ORIGIN" \
  -d '{"client_id":"finalapproval-cli"}')

DEVICE_CODE=$(echo "$RESP" | jq -r .device_code)
URL=$(echo "$RESP" | jq -r .verification_uri_complete)
INTERVAL=$(echo "$RESP" | jq -r .interval)

# Always print the URL so the developer can copy-paste if auto-open fails.
echo "Open this URL to approve: $URL"

# Always try to auto-open a browser. Best-effort — never fail the flow if it doesn't work.
( xdg-open "$URL" >/dev/null 2>&1 \
  || open "$URL" >/dev/null 2>&1 \
  || python3 -m webbrowser "$URL" >/dev/null 2>&1 \
  || true ) &
```

The browser open is best-effort. The URL is printed regardless — developer can copy-paste on headless machines or if the opener fails. Do not pause, do not prompt "press enter when ready". Continue directly to 2c.

**2c. Poll for the token (never pause, never prompt):**

Start polling immediately. The loop below handles all RFC 8628 states without any user interaction — the developer signs in in the browser whenever they're ready, and the next poll picks up the new token.

```bash
while true; do
  RESP=$(curl -s -X POST "$FINALAPPROVAL_URL/api/auth/device/token" \
    -H "Content-Type: application/json" \
    -H "Origin: $ORIGIN" \
    -d "{
      \"grant_type\":\"urn:ietf:params:oauth:grant-type:device_code\",
      \"device_code\":\"$DEVICE_CODE\",
      \"client_id\":\"finalapproval-cli\"
    }")

  ERR=$(echo "$RESP" | jq -r '.error // empty')
  case "$ERR" in
    authorization_pending) sleep "$INTERVAL" ;;
    slow_down)             INTERVAL=$((INTERVAL * 2)); sleep "$INTERVAL" ;;
    expired_token|access_denied)
      echo "Device code $ERR — ask the developer to retry."; exit 1 ;;
    "")
      TOKEN=$(echo "$RESP" | jq -r .access_token)
      mkdir -p ~/.finalapproval
      echo "{\"token\":\"$TOKEN\"}" > ~/.finalapproval/token.json
      chmod 600 ~/.finalapproval/token.json
      break ;;
    *)
      echo "Unexpected error: $ERR"; exit 1 ;;
  esac
done
```

Never insert a `read`, `wait`, or "press any key" step here. The poll loop IS the wait — it resolves the moment the developer finishes signing in.

All future API calls in this skill use `Authorization: Bearer $TOKEN`.

### 3. Lock in the webhook URL

You already know the choice from Q3 in step 1. Resolve it to a concrete URL:

| Choice from Q3 | Action |
|---|---|
| **(a)** Public URL provided | Use it directly. Verify it's HTTPS. |
| **(b)** Local + tunnel | Run `ngrok http 3000` (or `cloudflared tunnel --url http://localhost:3000`), capture the `https://*.ngrok-free.app` URL, append `/webhooks/finalapproval`. Tell them they can update to the production URL later from channel settings — the channel survives the swap. |
| **(c)** Scaffold Express | Use `http://localhost:3000/webhooks/finalapproval` only as a placeholder; immediately set up a tunnel (ngrok) to expose it. Without a public URL the channel can't deliver. |
| **(d)** Serverless | Deploy the route first (Vercel `app/api/webhooks/finalapproval/route.ts`, Cloudflare Worker, etc.), capture the deployed URL, use that. |

**Don't proceed until you have a publicly reachable HTTPS URL.** It's part of the channel from day one — polling is not an option.

### 4. Create the channel

```bash
TOKEN=$(jq -r .token ~/.finalapproval/token.json)

curl -s -X POST "$FINALAPPROVAL_URL/api/channels" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Send Email",
    "description": "Approve outgoing emails before sending",
    "webhook_url": "https://your-server.com/webhooks/finalapproval"
  }'
```

Response includes:
- `api_key` (`fa_...`) — for submitting approvals. **Shown once.**
- `webhook_secret` (`whsec_...`) — for verifying webhook signatures. Only returned when `webhook_url` is provided. **Shown once.**

### 5. Save channel credentials

The bearer token already lives at `~/.finalapproval/token.json`. The channel-scoped `fa_` key and webhook secret belong in the **project's** `.env`:

Check the developer's `.env` first — they may already have `FINALAPPROVAL_API_KEY` from a previous channel. Each channel gets its own key, so use a channel-specific name if multiple channels exist (e.g. `FINALAPPROVAL_EMAIL_API_KEY`).

Add any new credentials:

```
FINALAPPROVAL_API_KEY=fa_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
FINALAPPROVAL_WEBHOOK_SECRET=whsec_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

Both credentials are shown once and cannot be retrieved again. If lost, the API key requires creating a new channel; the webhook secret can be regenerated by updating the webhook URL in channel settings.

### 5.5. Seed a sample approval so the developer can test the UI immediately

**Always do this, even before any code is wired up.** The goal is to let the developer see what an approval looks like in the dashboard and practise the approve/deny flow without having to build the submission function first.

Using the `fa_` key just minted, post one sample approval. **Use the actual key returned from step 3** — don't leave a placeholder in the curl.

```bash
# $API_KEY must be the real fa_... value returned by step 3's channel-create response.
# Do NOT paste a literal "fa_..." — the request will 401.

curl -s -X POST "$FINALAPPROVAL_URL/api/v1/approvals" \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Sample approval — try approving or denying this",
    "body": "<div class=\"space-y-3\"><div class=\"rounded-lg border p-3\"><p class=\"text-xs text-gray-500\">This is a test</p><p class=\"font-medium\">A real approval from your agent will look like this card. Click Approve or Deny in the dashboard to see the resolution flow — no webhook is required for this test.</p></div><details class=\"rounded-lg border\"><summary class=\"cursor-pointer p-3 font-medium text-sm\">What happens next?</summary><div class=\"border-t p-3 text-sm text-gray-600\">Once you wire up the submission function (step 5), every call will render here. If you configure a webhook (step 6), your code runs the moment you click Approve.</div></details></div>",
    "data": { "sample": true, "channel": "'"$CHANNEL_NAME"'" }
  }'
```

Tell the developer: **open the channel in the dashboard now and click Approve or Deny on the sample card.** This validates end-to-end that the channel works, the body renders correctly, and the resolution UI behaves as expected — all before writing any integration code.

Resolving the sample also exercises the webhook delivery path you configured in step 3 — useful for confirming end-to-end connectivity once step 7 is wired up.

### 6. Build the approval submission function

This is the core integration. Create a single function that:
1. Builds a consistent HTML body from the action's runtime data
2. Includes the structured data for programmatic use in the webhook
3. Posts to the FinalApproval API

**The HTML body is a template defined once in code.** Every approval in this channel will look the same — only the data values change. Design the template for the specific use case identified in step 1.

```typescript
// lib/approval.ts — or wherever makes sense in the project

interface EmailApprovalData {
  recipient: string;
  subject: string;
  body: string;
  priority: "low" | "normal" | "high";
}

async function submitForApproval(data: EmailApprovalData): Promise<string> {
  const priorityColors = {
    low: { bg: "bg-gray-100", text: "text-gray-700" },
    normal: { bg: "bg-blue-100", text: "text-blue-700" },
    high: { bg: "bg-red-100", text: "text-red-700" },
  };
  const pc = priorityColors[data.priority];

  const htmlBody = `
    <div class="space-y-3">
      <div class="grid grid-cols-2 gap-3">
        <div class="rounded-lg border p-3">
          <p class="text-xs text-gray-500">Recipient</p>
          <p class="font-semibold">${data.recipient}</p>
        </div>
        <div class="rounded-lg border p-3">
          <p class="text-xs text-gray-500">Priority</p>
          <span class="inline-flex items-center rounded-full ${pc.bg} px-2 py-1 text-xs font-medium ${pc.text}">${data.priority}</span>
        </div>
      </div>
      <div class="rounded-lg border p-3">
        <p class="text-xs text-gray-500 mb-1">Subject</p>
        <p class="font-medium">${data.subject}</p>
      </div>
      <details class="rounded-lg border">
        <summary class="cursor-pointer p-3 font-medium text-sm">Email Body Preview</summary>
        <div class="border-t p-3 text-sm text-gray-600">${data.body}</div>
      </details>
    </div>`;

  const baseUrl = process.env.FINALAPPROVAL_URL ?? "https://www.finalapproval.ai";
  const response = await fetch(`${baseUrl}/api/v1/approvals`, {
    method: "POST",
    headers: {
      "Authorization": `Bearer ${process.env.FINALAPPROVAL_API_KEY}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      title: `Send email to ${data.recipient}`,
      body: htmlBody,
      data,
    }),
  });

  if (!response.ok) throw new Error("Failed to submit approval");
  const { id } = await response.json();
  return id; // "pending" — human reviews in dashboard
}
```

**Key rules:**
- `body` is the HTML template (what the human sees). `data` is structured JSON (what the webhook returns to your code). Always include both.
- The HTML body is sanitized by DOMPurify on the frontend. Only use allowed tags and attributes (see appendix).
- Design the template so a human can make a confident approve/deny decision from the rendered card alone.

#### Calling the function

At the point in the codebase where the action is triggered, replace the direct action with the approval submission:

```typescript
// Before: sendEmail(recipient, subject, body)
// After:
const approvalId = await submitForApproval({
  recipient,
  subject,
  body: emailBody,
  priority: "high",
});
// Action is now pending — human reviews in dashboard
// The webhook (step 7) fires when the human decides and runs the real action
```

### 7. Wire up the webhook receiver — close the loop

This is the half of the integration that actually *executes the gated action*. Without it, the submission function in step 6 just creates pending requests that nothing acts on. We don't skip this — ever.

When a human approves or denies, FinalApproval POSTs the decision to the webhook URL configured in step 3. **This is where the gated action actually runs.**

First, search the codebase for existing webhook handlers:
- Look for routes like `/webhooks`, `/api/hooks`, or similar patterns
- Check for existing Express/Fastify/Next.js/Hono API routes that handle incoming POSTs
- If a handler exists, add a new route for FinalApproval alongside it — match the existing patterns (router, middleware, error handling)

If no webhook infrastructure exists, scaffold a receiver based on what was decided in step 3:

- **Existing server with public URL** — add a new route (snippet below)
- **Local dev with tunnel (ngrok/cloudflared)** — add the same route to your local server; the tunnel forwards traffic to it
- **Serverless** — use the platform's HTTP handler signature (Vercel `app/api/webhooks/finalapproval/route.ts`, Cloudflare Worker `fetch()`, Lambda Function URL handler). The verification logic is identical; only the request/response shape differs.
- **No HTTP at all** — stand up a 20-line Express server in the same project just for this. The function that runs on approval can call into the same module the agent uses.

Express scaffold:

```typescript
import crypto from "node:crypto";

// Signature verification — MUST be done before trusting the payload
function verifyWebhook(headers: Record<string, string>, body: string): boolean {
  const secret = process.env.FINALAPPROVAL_WEBHOOK_SECRET!;
  const signature = headers["x-finalapproval-signature-256"];
  const timestamp = headers["x-finalapproval-timestamp"];

  // Reject replays older than 5 minutes
  const age = Math.abs(Date.now() / 1000 - Number(timestamp));
  if (age > 300) return false;

  const expected = "sha256=" + crypto
    .createHmac("sha256", secret)
    .update(`${timestamp}.${body}`)
    .digest("hex");

  return crypto.timingSafeEqual(
    Buffer.from(signature),
    Buffer.from(expected),
  );
}

// Webhook route
app.post("/webhooks/finalapproval", express.raw({ type: "application/json" }), (req, res) => {
  const rawBody = req.body.toString();

  if (!verifyWebhook(req.headers as Record<string, string>, rawBody)) {
    res.status(403).send("Invalid signature");
    return;
  }

  const { event, approval } = JSON.parse(rawBody);

  if (event === "approval.resolved") {
    if (approval.status === "approved") {
      // Execute the gated action using approval.data
      const { recipient, subject, body } = approval.data;
      sendEmail(recipient, subject, body); // <-- the action that was waiting
    } else {
      // Denied — log it, notify the agent, or take corrective action
      console.log(`Denied: ${approval.title}`);
      if (approval.denial_reason) {
        console.log(`Reason: ${approval.denial_reason}`);
      }
    }
  }

  res.status(200).send("ok");
});
```

**The webhook handler closes the loop.** The submission function (step 6) sends the request; the webhook handler (this step) receives the decision and runs the action. Both use `approval.data` as the shared contract — the structured JSON that carries the runtime values through the approval flow.

**Both the approve AND deny paths must be handled.** Approving without handling denials means denied requests silently disappear — the agent has no idea its request was rejected. At minimum, log denials with their `denial_reason`. Better: surface them back to the caller (return an error, write to a status table, notify the agent's session) so the agent can adapt — retry with corrections, escalate, or abandon the task.

#### Idempotency

FinalApproval may retry webhook deliveries on failure (network blips, 5xx responses). Make the handler idempotent: track which `approval.id`s you've already processed (a Set in memory for ephemeral handlers, a DB row for persistent ones) and skip duplicates. Otherwise an approved deploy could run twice.

#### Webhook payload schema

```json
{
  "event": "approval.resolved",
  "timestamp": "2026-04-11T12:00:00.000Z",
  "approval": {
    "id": "uuid",
    "channel_id": "uuid",
    "title": "Send email to alice@example.com",
    "body": "<div class=\"space-y-3\">...</div>",
    "data": { "recipient": "alice@example.com", "subject": "...", "body": "..." },
    "status": "approved",
    "denial_reason": null,
    "created_at": "2026-04-11T11:55:00.000Z",
    "resolved_at": "2026-04-11T12:00:00.000Z",
    "resolved_by": "user_abc123"
  },
  "channel": { "id": "uuid", "name": "Send Email" }
}
```

Security headers on every delivery:
- `X-FinalApproval-Signature-256: sha256=<hmac_hex>` — HMAC-SHA256 of `timestamp.body`
- `X-FinalApproval-Timestamp: <unix_seconds>` — reject if older than 5 minutes

### 8. Verify the full loop end-to-end

Walk the developer through confirmation. **All of these must pass — the loop isn't closed until they do.**

1. **Channel is live** — visible at the FinalApproval dashboard
2. **Submission works** — trigger the action in the codebase, confirm it appears as a pending approval in the dashboard
3. **Body renders correctly** — the HTML template displays the runtime data clearly in the approval card
4. **Webhook fires on approval** — approve the request in the dashboard, confirm the webhook handler receives it (check logs) and executes the action (the email actually sends, the deploy actually runs)
5. **Webhook fires on denial** — deny a request with a reason, confirm the handler receives `denial_reason` and the action is *not* executed
6. **Test from settings** — open the channel settings in the dashboard and click "Test" to send a synthetic webhook delivery. Confirm signature verification passes

If any step fails, check:
- `~/.finalapproval/token.json` exists and the token still works (re-run step 2 if not)
- `FINALAPPROVAL_API_KEY` is set in the project's `.env` (starts with `fa_`)
- Webhook URL is reachable from the FinalApproval server
- Webhook secret matches (regenerate by updating the webhook URL in channel settings)
- Signature verification uses the raw body string, not a re-serialized object

---

## Appendix: HTML Sanitization Rules

**Allowed tags:** `div`, `span`, `p`, `h1`-`h6`, `ul`, `ol`, `li`, `dl`, `dt`, `dd`, `table`, `thead`, `tbody`, `tr`, `th`, `td`, `strong`, `em`, `small`, `code`, `pre`, `blockquote`, `a`, `br`, `hr`, `details`, `summary`, `img`, `figure`, `figcaption`

**Allowed attributes:** `class`, `href`, `src`, `alt`, `target`, `rel`, `loading`

**Images:** `src` must use `https://`. Other schemes (`http://`, `data:`, relative paths) are stripped at render time.

**Links:** Automatically get `target="_blank" rel="noopener noreferrer"`.

**Stripped silently:** `<script>`, `<iframe>`, `<form>`, `<input>`, `<video>`, `<svg>`, `style` attribute, `on*` handlers, `id`, `data-*`.
