# AI Customer Support Ticket System — Implementation Guide

A production-style n8n workflow that reads support emails from Gmail, classifies and drafts a reply with Groq (`llama-3.1-8b-instant`), routes the draft through a Slack human-in-the-loop approval, and logs every ticket to Google Sheets — with retries, rate-limit handling, and a dedicated error-logging workflow.

Two files ship alongside this guide, ready to import into n8n:
- `AI-Support-Ticket-System.json` — the main workflow
- `AI-Support-Ticket-Error-Handler.json` — the error-handling workflow

---

## 1. Prerequisites

| Item | Where to get it |
|---|---|
| Self-hosted n8n instance | `docker run -it --rm -p 5678:5678 n8nio/n8n` or your existing install |
| Groq API key | console.groq.com → API Keys |
| Gmail OAuth2 credential | Google Cloud Console → enable Gmail API → OAuth consent screen → n8n's built-in Gmail node credential flow |
| Slack Bot token | api.slack.com/apps → create app → OAuth scopes: `chat:write`, `channels:read`, `users:read` |
| Google Sheets OAuth2 credential | Google Cloud Console → enable Sheets API → n8n's Google Sheets credential flow |

---

## 2. Security setup (do this first)

Never paste API keys into node parameters — every credential in the JSON files below is referenced by a **credential name**, not a raw key.

**Groq API key** — n8n has no native "Groq" credential type, so store it as a **Header Auth credential**:
1. n8n → Credentials → New → *Header Auth*
2. Name: `Groq API Key (Authorization: Bearer)`
3. Header name: `Authorization`
4. Header value: `Bearer YOUR_GROQ_API_KEY`

This keeps the key out of the workflow JSON entirely — the JSON only stores a credential ID reference.

**Environment variables** — for anything referenced outside a credential (Sheet ID, Slack channel IDs, etc.), set them on the n8n host and pull them in with `{{ $env.VAR_NAME }}`:
```bash
# .env or docker-compose environment block
GOOGLE_SHEET_ID=1AbCdEfGhIjKlMnOpQrStUvWxYz
SLACK_APPROVAL_CHANNEL=support-approvals
SLACK_ALERT_CHANNEL=support-alerts
```
Then in any node parameter: `{{ $env.GOOGLE_SHEET_ID }}`. In the imported JSON, `GOOGLE_SHEET_ID` placeholders should be swapped for this expression (or just pasted directly if you prefer per-workflow config — env vars are the more "production" choice for a portfolio writeup).

**Validate AI responses before parsing** — handled by the `Validate & Parse Classification` Code node (see §5) which strips stray code fences, checks the JSON shape, and clamps any out-of-vocabulary category/urgency/sentiment values back to a safe default instead of crashing the workflow.

---

## 3. Architecture

The diagram above shows the full path: Gmail Trigger → Extract & build ticket → Groq classify → validate/parse → Groq draft → Slack approval → branch on approve/reject → Gmail send + Sheets log, or Slack notify + Sheets log. A separate Error Handler workflow (§9) is wired in via **Workflow Settings → Error Workflow**, so any node failure in the main flow is caught centrally.

---

## 4. Node-by-node reference (main workflow)

| # | Node | Type | Purpose |
|---|---|---|---|
| 1 | Gmail Trigger | `gmailTrigger` | Polls inbox every minute for unread mail |
| 2 | Extract Email Data | `Set` | Pulls sender name/email, subject, body, timestamp; generates Ticket ID |
| 3 | Groq - Classify Email | `HTTP Request` | Calls Groq chat completions for category/urgency/sentiment/summary |
| 4 | Validate & Parse Classification | `Code` | Parses + validates the JSON, clamps invalid values, throws on garbage |
| 5 | Groq - Generate Draft Reply | `HTTP Request` | Calls Groq to draft the customer-facing reply |
| 6 | Extract Draft Text | `Code` | Pulls the reply text out of the Groq response, throws if empty |
| 7 | Slack - Send for Approval | `Slack` (Send and Wait) | Posts ticket + draft, pauses workflow until a human clicks Approve/Reject |
| 8 | Approved? | `If` | Branches on the boolean returned by the Slack approval webhook |
| 9a | Gmail - Send Reply | `Gmail` | Sends the approved draft as a threaded reply |
| 10a | Sheets - Log Approved Ticket | `Google Sheets` | Appends the ticket row with status "Approved - Sent" |
| 9b | Slack - Notify Rejection | `Slack` | Posts a message asking a human to edit and send manually |
| 10b | Sheets - Log Rejected Ticket | `Google Sheets` | Appends the ticket row with status "Rejected - Needs Manual Edit" |

### Node 1 — Gmail Trigger
- **Poll time:** every minute
- **Filters:** `labelIds: INBOX`, `readStatus: unread`
- Credential: Gmail OAuth2 (support inbox)

### Node 2 — Extract Email Data (Set node)
Generates:
- `ticketId` — `TCK-YYYYMMDD-HHMMSS-###`
- `senderName` / `senderEmail` — parsed out of the raw `From` header
- `subject`, `body`, `receivedAt`
- `gmailMessageId`, `gmailThreadId` — needed later so the reply threads correctly

### Node 3 — Groq - Classify Email (HTTP Request)
See exact configuration in §5.

### Node 4 — Validate & Parse Classification (Code)
See exact logic in §6 — this is the "validate AI responses before parsing" security requirement in action.

### Node 5 — Groq - Generate Draft Reply (HTTP Request)
See exact configuration in §5.

### Node 6 — Extract Draft Text (Code)
Pulls `choices[0].message.content` from the Groq draft response and throws a descriptive error if it's missing/empty (this is what the error workflow's Slack alert will show).

### Node 7 — Slack - Send for Approval
Uses n8n's built-in **Slack "Send and Wait for Approval"** operation (`resource: message`, `operation: sendAndWait`). This is the human-in-the-loop core of the whole project:
- Posts the ticket summary + draft to `#support-approvals` with native Approve/Reject buttons
- n8n pauses the execution (no polling, no extra webhook to build by hand) until someone clicks a button
- `limitWaitTime`: auto-resumes (as rejected/timeout) after 24 hours so tickets never hang forever
- The resume payload lands in `$json.data.approved` (boolean), which node 8 reads directly

### Node 8 — Approved? (If)
Single condition: `{{ $json.data.approved }} === true`.

### Nodes 9a/10a — Approved branch
Gmail reply threads using `gmailThreadId` + `gmailMessageId` captured back in node 2, so the customer sees a normal reply in their inbox, not a new email. Sheets append writes the full row with `Response Status = "Approved - Sent"`.

### Nodes 9b/10b — Rejected branch
Slack posts to the team channel asking for manual handling. Sheets append writes the row with `Response Status = "Rejected - Needs Manual Edit"` and blank `Final Response` / `Resolution Date` — these get filled in by hand (or by a small follow-up automation) once a human sends the edited reply.

---

## 5. Groq HTTP Request node configurations (exact)

Both Groq calls use the same base setup:

- **Method:** `POST`
- **URL:** `https://api.groq.com/openai/v1/chat/completions`
- **Authentication:** Generic Credential Type → Header Auth → `Groq API Key (Authorization: Bearer)`
- **Headers:** `Content-Type: application/json`
- **Body:** JSON (Specify Body → JSON)
- **Options → Retry on Fail:** enabled, `maxTries: 4`, `waitBetweenTries: 3000ms` — this is what covers **429 rate-limit retries** without any custom loop code. n8n retries the same request up to 4 times with a 3-second gap; Groq's `Retry-After` header combined with this backoff is normally enough headroom on the free/dev tier.
- **On Error:** `Continue (using error output)` — so a fully-exhausted retry doesn't crash silently; it routes to the connected error path / triggers the Error Workflow via `execution.error`.

### 5.1 Classification call — request body
```json
{
  "model": "llama-3.1-8b-instant",
  "temperature": 0.2,
  "max_tokens": 300,
  "response_format": { "type": "json_object" },
  "messages": [
    { "role": "system", "content": "<see prompt in §7.1>" },
    { "role": "user", "content": "<see prompt template in §7.1>" }
  ]
}
```
`response_format: json_object` is Groq/OpenAI-compatible "JSON mode" — it constrains the model to emit syntactically valid JSON, which is the first line of defense before the Code node's manual validation.

### 5.2 Draft-generation call — request body
```json
{
  "model": "llama-3.1-8b-instant",
  "temperature": 0.5,
  "max_tokens": 350,
  "messages": [
    { "role": "system", "content": "<see prompt in §7.2>" },
    { "role": "user", "content": "<see prompt template in §7.2>" }
  ]
}
```
No `response_format` here since the output is meant to be plain-text email copy, not JSON.

---

## 6. Validation logic (Code node, node 4)

Key behaviors:
- Strips accidental ` ```json ` fences if the model adds them despite instructions
- `JSON.parse`s the cleaned string in a try/catch
- Throws a descriptive `Error` (caught by the Error Workflow) if parsing fails or content is missing
- Clamps `category` / `urgency` / `sentiment` to the allowed enum lists, defaulting to `General Inquiry` / `Medium` / `Neutral` if the model returns something unexpected — this means one bad-but-valid-JSON response degrades gracefully instead of breaking the run

---

## 7. Exact prompts

### 7.1 Email classification

**System prompt:**
```
You are an AI assistant that classifies customer support emails for a SaaS company. Respond ONLY with a single valid JSON object. No markdown, no code fences, no explanation, no text outside the JSON. The JSON must exactly match this schema: {"summary": string (max 2 sentences, plain text), "category": one of ["Bug", "Feature Request", "Billing", "General Inquiry"], "urgency": one of ["Low", "Medium", "High"], "sentiment": one of ["Positive", "Neutral", "Negative"]}. If unsure, choose the closest matching category and default urgency to Medium.
```

**User prompt template:**
```
Sender: {{ senderName }} <{{ senderEmail }}>
Subject: {{ subject }}
Body:
{{ body }}

Analyze this email and return the JSON classification only.
```

### 7.2 Draft reply generation

**System prompt:**
```
You are a professional, empathetic customer support agent. Write email replies that are warm, concise, and under 150 words. Do not promise specific fixes, timelines, or new features. When appropriate, mention general next steps such as 'our team is reviewing this'. Return ONLY the plain-text email body, no subject line, no markdown, ending with 'Best regards,\nSupport Team'.
```

**User prompt template:**
```
Customer name: {{ senderName }}
Subject: {{ subject }}
Original message:
{{ body }}

Classification -> category: {{ category }}, urgency: {{ urgency }}, sentiment: {{ sentiment }}
Summary: {{ summary }}

Write a professional reply email body.
```

---

## 8. Sample data

### 8.1 Sample Gmail trigger output (raw, before extraction)
```json
{
  "id": "18d2f9a7b4e6c3a1",
  "threadId": "18d2f9a7b4e6c3a1",
  "from": "Maria Santos <maria.santos@example.com>",
  "subject": "App crashes every time I try to export a report",
  "date": "2026-07-05T09:14:22Z",
  "text": "Hi team, every time I click 'Export to PDF' on the monthly report page the app freezes and I have to force-close it. This has happened 5 times today on Chrome and once on Safari. I need this report for a meeting tomorrow morning. Please help urgently."
}
```

### 8.2 Sample extracted ticket object (after node 2 + 4 + 6)
```json
{
  "ticketId": "TCK-20260705-091430-582",
  "senderName": "Maria Santos",
  "senderEmail": "maria.santos@example.com",
  "subject": "App crashes every time I try to export a report",
  "body": "Hi team, every time I click 'Export to PDF' on the monthly report page the app freezes...",
  "receivedAt": "2026-07-05T09:14:22Z",
  "gmailMessageId": "18d2f9a7b4e6c3a1",
  "gmailThreadId": "18d2f9a7b4e6c3a1",
  "summary": "Customer reports the app freezes every time they try to export a monthly report to PDF, across Chrome and Safari.",
  "category": "Bug",
  "urgency": "High",
  "sentiment": "Negative",
  "draftResponse": "Hi Maria,\n\nThank you for reaching out, and I'm sorry for the frustration this is causing, especially with your meeting coming up. I've logged the details of the PDF export freeze you're experiencing on both Chrome and Safari and passed it to our engineering team for review.\n\nIn the meantime, if you're able to export in a different format (e.g. CSV) that may work as a temporary workaround for tomorrow's meeting. We'll follow up as soon as we have more information.\n\nThanks for your patience.\n\nBest regards,\nSupport Team"
}
```

### 8.3 Sample Slack approval message
```
New support ticket needs approval 📩

Ticket: TCK-20260705-091430-582
Customer: Maria Santos (maria.santos@example.com)
Subject: App crashes every time I try to export a report
Category: Bug   Urgency: High   Sentiment: Negative

AI Summary:
Customer reports the app freezes every time they try to export a monthly report to PDF, across Chrome and Safari.

Draft Response:
Hi Maria,

Thank you for reaching out, and I'm sorry for the frustration this is causing...

[ Approve & Send ]   [ Reject ]
```

### 8.4 Sample Slack rejection notice
```
❌ Ticket TCK-20260705-091430-582 was rejected

Customer: Maria Santos (maria.santos@example.com)
Subject: App crashes every time I try to export a report

The AI draft was not approved. Please edit the response manually and send it, then update the ticket status in the sheet.
```

### 8.5 Sample Slack error alert
```
🚨 Workflow failure detected

Error ID: ERR-20260705-101502
Workflow: AI Customer Support Ticket System
Failed node: Groq - Classify Email
Rate limited (429): Yes - Groq API throttled, retries were exhausted
Message: Request failed with status code 429

View execution
```

---

## 9. Google Sheets structure

**Sheet name:** `Tickets`

| Ticket ID | Date | Customer Name | Customer Email | Subject | Category | Sentiment | Urgency | AI Summary | Draft Response | Response Status | Final Response | Resolution Date |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| TCK-20260705-091430-582 | 2026-07-05T09:14:22Z | Maria Santos | maria.santos@example.com | App crashes every time... | Bug | Negative | High | Customer reports the app freezes... | Hi Maria, Thank you for reaching out... | Approved - Sent | Hi Maria, Thank you for reaching out... | 2026-07-05T09:16:40Z |

**Sheet name:** `Error Log` (second tab in the same spreadsheet, used by the error workflow)

| Error ID | Timestamp | Workflow | Failed Node | Error Message | Is Rate Limit | Execution URL |
|---|---|---|---|---|---|---|
| ERR-20260705-101502 | 2026-07-05T10:15:02Z | AI Customer Support Ticket System | Groq - Classify Email | Request failed with status code 429 | true | https://your-n8n.host/workflow/.../executions/1234 |

---

## 10. Error handling workflow (`AI-Support-Ticket-Error-Handler.json`)

This is a **separate importable workflow**, linked to the main one via **Main workflow → Settings → Error Workflow → "AI Support Ticket - Error Handler"**. n8n automatically calls it whenever any node in the main workflow throws (including exhausted-retry HTTP failures), passing execution + error context.

| Node | Type | Purpose |
|---|---|---|
| Error Trigger | `Error Trigger` | Entry point, fires on any failed execution of the linked workflow |
| Format Error Data | `Set` | Builds a clean error record, detects 429/rate-limit via string match on the error message |
| Sheets - Log Error | `Google Sheets` | Appends to the `Error Log` tab |
| Slack - Alert Team | `Slack` | Posts to `#support-alerts` with a link to the failed execution |

**429-specific handling layers (defense in depth):**
1. HTTP Request node's built-in `retryOnFail` (4 tries, 3s apart) — usually enough to ride out a short-lived Groq rate limit.
2. `onError: continueErrorOutput` on both Groq nodes — if retries are exhausted, the error is surfaced instead of silently stalling.
3. Error Trigger workflow — logs it and pages the team via Slack so a human knows Groq is throttling and can investigate (e.g. check usage tier, add backoff, or pause the trigger temporarily).

---

## 11. Production best practices demonstrated in this project

- **Least-privilege credentials** — separate Header Auth credential for Groq, OAuth2 scoped credentials for Gmail/Sheets, dedicated Slack bot token; nothing hardcoded in node parameters.
- **Defense-in-depth error handling** — per-node retry, per-node error routing, plus a centralized Error Workflow (mirrors how you'd wire up alerting in a real SRE setup, e.g. Sentry + PagerDuty pattern).
- **AI output validation** — never trust an LLM to always return clean JSON; parse defensively, clamp to an enum, fail loudly (not silently) on garbage.
- **Human-in-the-loop by design** — no automated email ever goes out without explicit human approval, using n8n's native Slack "Send and Wait" pattern rather than a hand-rolled webhook + database polling loop.
- **Idempotent, traceable records** — every ticket gets a unique, sortable Ticket ID and is logged regardless of outcome (approved or rejected), giving a full audit trail.
- **Threaded replies, not new emails** — using the original Gmail `threadId`/`messageId` so customers see a normal conversation, not a disconnected new message.
- **Timeouts on human steps** — the Slack approval step auto-resumes after 24 hours so a forgotten Slack message can't leave a ticket stuck forever.
- **Environment-variable-driven config** — Sheet IDs and Slack channel names are designed to be pulled from `$env` rather than pasted into every node, so the same workflow JSON can move between dev/staging/production by changing environment variables only.

---

## 12. Import steps

1. In n8n: **Workflows → Import from File** → select `AI-Support-Ticket-Error-Handler.json`. Save it, then note its workflow ID.
2. Import `AI-Support-Ticket-System.json` the same way.
3. In the main workflow's **Settings**, set **Error Workflow** to the error handler you just imported.
4. Recreate the four credentials listed in §1/§2 (Gmail OAuth2, Groq Header Auth, Slack Bot token, Google Sheets OAuth2) and re-attach them to each node — credentials are never exported in workflow JSON for security reasons, so this step is always manual after import.
5. Replace the `GOOGLE_SHEET_ID` placeholder in both workflows with your real Sheet ID (or an `{{ $env.GOOGLE_SHEET_ID }}` expression).
6. Replace the Slack channel names (`support-approvals`, `support-alerts`) with your actual channels, and invite the bot to both.
7. Create the two-tab Google Sheet (`Tickets`, `Error Log`) with the headers from §9.
8. Send yourself a test email to the connected inbox and watch the execution log step through classify → draft → Slack approval → your decision → Gmail send / Sheets log.

---

## 13. What to say about this project in an interview / portfolio writeup

This project is a good talking point because it touches: webhook/polling triggers, prompt engineering with structured JSON output, defensive parsing of untrusted LLM output, human-in-the-loop approval UX (not just "call an API and hope"), multi-branch conditional logic, cross-service API integration (Gmail, Slack, Sheets, Groq), and — critically — a *separate, first-class error-handling workflow* rather than bolting error handling onto the happy path. That last point is what tends to separate a demo from something that looks production-ready.
