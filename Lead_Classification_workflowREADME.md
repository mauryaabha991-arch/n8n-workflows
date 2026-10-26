# Lead Classification Workflow (n8n)

An n8n workflow that receives lead data through a webhook, validates it, scores it, uses AI to read the buying intent from the notes, classifies each lead as **Hot**, **Warm**, or **Cold**, saves it to Google Sheets, and emails an alert for every Hot lead.

## Features

- Webhook intake for `name`, `email`, `phone`, `budget`, `notes`
- Data cleaning and validation (missing budget, invalid email, currency symbols like `₹60,000`)
- Multi-tier classification using **IF** and **Switch** nodes
- 1-10 lead score based on budget, urgency keywords, and data completeness
- AI intent analysis of the lead notes (Google Gemini via an AI Agent node)
- Google Sheets storage with a `Category` column
- Gmail alert for every Hot lead

## Workflow overview

```
Webhook → Code (clean, validate, score) ─┬→ AI Agent (intent) ─┐
                                         └─────────────────────┴→ MergeAI → If (incomplete?)
                                                                              │
                              true  → Edit Fields (Cold) ─────────────────────┤
                              false → Switch ─ Hot  → Edit Fields1 (Hot)  ────┤
                                              Warm → Edit Fields2 (Warm) ────┼→ Merge → Append row in sheet → Respond to Webhook
                                              Else → Edit Fields (Cold)  ────┤        └→ If1 (Hot?) → Gmail alert
```

## Classification rules

| Category | Condition |
|---|---|
| **Hot** | Budget > 50,000, **or** notes contain a hot keyword (`urgent`, `immediate`, `immediately`, `asap`), **or** AI intent is `High` |
| **Warm** | Budget from 10,000 to 50,000 (inclusive) |
| **Cold** | Budget < 10,000, **or** missing or invalid critical fields (name, email, budget) |

Order matters. Incomplete data is checked first (IF node), then Hot, then Warm. Anything else is Cold.

### Edge cases handled

| Input | Result |
|---|---|
| Budget missing or `"abc"` | Cold, `issues` = `missing/invalid budget` |
| Budget `"₹60,000"` | Cleaned to `60000`, Hot |
| Empty or invalid email | Cold, `issues` = `missing/invalid email` |
| Empty name | Cold, `issues` = `missing name` |
| Budget exactly 50,000 | Warm |
| Budget exactly 10,000 | Warm |
| AI call fails | Workflow continues; intent falls back to `N/A` |

## Lead score (1-10)

| Criterion | Points |
|---|---|
| Budget ≥ 100,000 | +5 |
| Budget > 50,000 | +4 |
| Budget 10,000 - 50,000 | +3 |
| Budget > 0 and < 10,000 | +1 |
| Hot keyword in notes | +2 |
| Valid email | +1 |
| Phone with 10+ digits | +1 |
| Notes longer than 20 characters | +1 |

The score is clamped between 1 and 10. Weights are in the **Code in JavaScript** node.

## Setup

### 1. Import the workflow
In n8n: **Workflows → Import from File** and choose `lead-classification-workflow.json`.

### 2. Create the Google Sheet
Create a sheet with these headers in row 1 (exact names):

```
Name | Email | Phone | Budget | Notes | Issues | Category | Receive At | Scores | Intent
```

### 3. Connect credentials
The exported file contains **no credentials**. After importing, open each node and select or create your own:

| Node | Credential |
|---|---|
| Append row in sheet | Google Sheets OAuth2 |
| Send a message | Gmail OAuth2 |
| Google Gemini Chat Model | Google Gemini (PaLM) API key |

### 4. Replace the placeholders

| Node | Field | Replace with |
|---|---|---|
| Append row in sheet | Document | Your Google Sheet (the `YOUR_GOOGLE_SHEET_ID` placeholder) |
| Append row in sheet | Sheet | Your tab name (default `Sheet1`) |
| Send a message | To | The email that should receive Hot lead alerts (`your-email@example.com`) |
| Webhook | Path | Default is `lead-intake`. Use a long random path in production. |

### 5. Activate
Toggle the workflow to **Active** and use the **Production URL** from the Webhook node.

## Usage

Send a `POST` request with a JSON body:

```bash
curl -X POST "https://YOUR-N8N-HOST/webhook/lead-intake" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "abha maurya",
    "email": "abha@example.com",
    "phone": "9876543210",
    "budget": 75000,
    "notes": "Need this urgent"
  }'
```

Response:

```json
{ "status": "success", "message": "Submitted successfully" }
```

While building, use the **Test URL** (`/webhook-test/...`) and click **Listen for test event** before each request.

## Test cases

| Payload | Expected category | Notes |
|---|---|---|
| budget 75000, "Need this urgent" | Hot | Alert email sent |
| budget 5000, "We want to start next week, please send a quote" | Hot | Hot through AI intent only |
| budget 25000, "just looking around" | Warm | Intent Low, no alert |
| budget 3000, "hello" | Cold | |
| no budget field | Cold | `issues` = `missing/invalid budget` |
| budget `"₹60,000"` | Hot | Cleaned to 60000 |

## Customizing

| What to change | Where |
|---|---|
| Hot keywords | `HOT_KEYWORDS` array in the Code node |
| Score weights | Score block in the Code node |
| Budget thresholds | Switch node rule expressions |
| Required fields | `issues` checks in the Code node |
| AI behavior | System message in the AI Agent node |
| Let AI decide only the label, not Hot | Remove the `.includes('high')` part from the Switch Hot rule |

## Security notes

- Never commit credentials, API keys, or your real Google Sheet ID to a public repo.
- n8n exports include credential IDs, your instance ID, and webhook paths. Review any new export before pushing.
- Anyone who knows your webhook URL can post leads to it. Use a hard-to-guess path, or add header authentication on the Webhook node.
- Lead data contains personal information (names, emails, phone numbers). Do not push real leads or screenshots of them.

## Tech stack

n8n, Google Sheets, Gmail, Google Gemini, JavaScript (Code node)


