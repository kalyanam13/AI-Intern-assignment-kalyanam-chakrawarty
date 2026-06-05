# AI Intern Assignment — HSA Team
**Kalyanam · kalyanamchakrawarty66@email.com · June 2026**

---

## Repository Structure

```
ai-intern-assignment-kalyanam/
├── part-a/
│   └── index.html                        ← Student lead capture form
├── part-b/
│   ├── workflow-b1-lead-notification.json ← N8N lead notification workflow
│   ├── workflow-b2-scheduled-fetch.json  ← N8N scheduled data fetch workflow
│   ├── screenshot-b1.png                 ← Canvas screenshot (add manually)
│   └── screenshot-b2.png                 ← Canvas screenshot (add manually)
├── part-c/
│   ├── index.html                        ← Integrated form with fetch()
│   └── loom-link.txt                     ← Screen recording link
└── README.md
```

---

## Part A — HTML/CSS/JS Lead Capture Form

**File:** `part-a/index.html`

A fully self-contained student lead capture form built with pure HTML, CSS, and vanilla JavaScript (no frameworks).

**Features:**
- Fields: Full name, Email, Country (dropdown), Course Level (radio: UG/PG/PhD), Preferred University, Message (with live 600-char counter)
- Client-side validation with inline error messages on each field
- Thank-you state shown without page reload on successful submit
- Form data logged as JSON to the browser console on submission
- Fully responsive — tested at 320px, 768px, and 1280px widths

**How to run:** Open `part-a/index.html` directly in any browser. No server needed.

---

## Part B — N8N Automation Workflows

### B1 — Lead Notification Workflow

**File:** `part-b/workflow-b1-lead-notification.json`

**Flow:**
1. **Webhook node** — listens for a POST request to `/hsa-lead` (simulating a form submission)
2. **Set node** — extracts and renames key fields: `name`, `email`, `country`, `courseLevel`, `university`, `message`, `submittedAt`
3. **IF node** — branches on `courseLevel` matching regex `^(PG|PhD)$`
   - **True branch** → **Email Send node** — sends a priority notification to the advisors inbox
   - **False branch** → **Google Sheets node** — appends the UG lead row to a tracking sheet
4. **Respond to Webhook node** — returns `{"status":"ok"}` to the caller

**To import:** N8N → Workflows → Import → select `workflow-b1-lead-notification.json`

**Credentials to configure:**
| Node | Credential | Placeholder in JSON |
|---|---|---|
| Send Email | SMTP Account | `REPLACE_WITH_YOUR_SMTP_CREDENTIAL_ID` |
| Google Sheets | Google Sheets OAuth2 | `REPLACE_WITH_YOUR_GOOGLE_SHEETS_CREDENTIAL_ID` |
| Google Sheets | Sheet ID | `REPLACE_WITH_YOUR_GOOGLE_SHEET_ID` |

> If you have no SMTP or Sheets access, swap those nodes with **No Operation** mock nodes for testing. The logic and branching remain identical.

---

### B2 — Scheduled University Data Fetch Workflow

**File:** `part-b/workflow-b2-scheduled-fetch.json`

**API Used:** [Hipolabs Universities API](http://universities.hipolabs.com/)

**Why this API?** It is completely free, requires no authentication key, and returns structured JSON data (university name, country, web pages, domains) that is directly relevant to HSA's core business — university research. It's also reliable and low-latency, making it ideal for a daily cron job.

**Flow:**
1. **Schedule/Cron node** — fires daily at 09:00
2. **HTTP Request node** — `GET http://universities.hipolabs.com/search?country=United+Kingdom`
3. **Code node** — JavaScript transformation:
   - Flattens and filters the response to only include universities with at least one listed web page
   - Sorts alphabetically by name
   - Limits to the top 50 entries
   - Maps each entry to clean fields: `universityName`, `country`, `countryCode`, `primaryWebsite`, `domain`, `fetchedAt`
4. **Set node** — outputs clean, labelled fields ready for downstream use (e.g., appending to a CRM sheet)

**To import:** N8N → Workflows → Import → select `workflow-b2-scheduled-fetch.json`

No credentials required — the Hipolabs API is public.

---

## Part C — Integration Challenge (Bonus)

**File:** `part-c/index.html`

**Screen recording:** See `part-c/loom-link.txt`

Extends the Part A form to POST JSON data to the B1 webhook via `fetch()`.

**Key implementation details:**
- Submit button shows a loading spinner and "Sending…" text while the request is in flight, then disables to prevent double-submission
- `Content-Type: application/json` header set on every request
- On success (HTTP 2xx): hides the form and displays the thank-you message
- On error (network failure or non-2xx): shows an inline error toast with the status code and message; re-enables the button for retry
- A **live flow indicator** panel animates through the N8N node sequence so the user can see what's happening in the pipeline
- The webhook URL is configurable via an on-page input (no need to edit source code for testing)

**How to test:**
1. Import and activate `workflow-b1-lead-notification.json` in N8N
2. Copy the production webhook URL from the Webhook node
3. Open `part-c/index.html` in a browser
4. Paste the webhook URL in the banner at the top
5. Fill and submit the form — watch the flow indicator and check N8N execution logs

---

## Challenges Faced & How They Were Resolved

**1. N8N Webhook responding with Respond to Webhook vs. Last Node:**
N8N's webhook by default returns the output of the last node only in "Last Node" response mode. Switching `responseMode` to `responseNode` and adding a dedicated **Respond to Webhook** node at the end let both branches (Email and Sheets) converge and return a consistent `{"status":"ok"}` JSON response — critical for Part C's `fetch()` error handling.

**2. Hipolabs API returning a flat array vs. N8N's item-based data model:**
N8N's HTTP Request node wraps the parsed JSON as `item.json`. When the response is a raw JSON array, it arrives as a single item whose `.json` property *is* the array rather than N8N splitting it into multiple items. The Code node therefore explicitly checks `Array.isArray(item.json)` and uses `flatMap` to normalise the structure before filtering, preventing a silent failure where zero items would pass through.

**3. Responsive layout with the radio group:**
The CSS `:has()` pseudo-class (used to style the checked radio label) is not supported in Firefox < 121. Added a JavaScript fallback that toggles a `.checked` class on the parent label on `change` events to ensure consistent visual feedback across all browsers.

---

## Environment Variables / Credential Placeholders

| Variable | Where to set | Notes |
|---|---|---|
| `REPLACE_WITH_YOUR_SMTP_CREDENTIAL_ID` | N8N credential panel | Any SMTP provider (Gmail, SendGrid, etc.) |
| `REPLACE_WITH_YOUR_GOOGLE_SHEETS_CREDENTIAL_ID` | N8N credential panel | OAuth2, requires Google Cloud project |
| `REPLACE_WITH_YOUR_GOOGLE_SHEET_ID` | B1 workflow JSON, Google Sheets node | Found in the Sheet URL |
| Webhook URL | `part-c/index.html` config banner | Copy from N8N Webhook node after activation |
