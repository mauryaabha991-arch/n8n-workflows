# Customer Query Auto-Reply with n8n + AI

An n8n automation that receives customer queries through a webhook, uses an AI agent to classify them and write a personalized reply, escalates urgent or unclear queries to the team, logs every query to Google Sheets, and sends a daily summary report.

## The Problem

Support teams get a constant stream of questions (pricing, refunds, help requests). Replying manually is slow, and urgent or angry customers can get lost in the pile.

This project:
1. Receives the query (name, email, message)
2. Detects the category and sentiment with AI
3. Replies with a relevant, personalized message
4. Alerts the team when a query is urgent, negative, or unrecognized
5. Logs everything in Google Sheets
6. Sends a daily report of query volumes

## Architecture

### Workflow 1: `Customer_query_reply`

```
Webhook (POST) -> AI Agent (GPT-4o-mini) -> Code (parse JSON) -> IF (escalate?)
                                                                  |-- true  -> Telegram (team alert) -> Telegram (customer reply) -> Google Sheets
                                                                  |-- false ----------------------->  Telegram (customer reply) -> Google Sheets
```

| Node | Purpose |
|---|---|
| Webhook | Receives `POST /customer_query` with name, email, message |
| AI Agent + OpenAI Chat Model | Returns JSON with `category`, `sentiment`, `priority`, `reply` |
| Code in JavaScript | Safely parses the AI output; falls back to a default reply if parsing fails |
| If | Escalates when priority is High, category is Other, or sentiment is Negative |
| teams_telegram | Alerts the team about escalated queries |
| customer_telegram | Sends the AI-written reply |
| company_sheet | Appends a row to Google Sheets |

### Workflow 2: `Daily_summary_report`

```
Schedule Trigger (daily) -> Google Sheets (read rows) -> Code (count by category) -> Telegram (report)
```

## Categories

| Category | Trigger | Reply content |
|---|---|---|
| Pricing | price, cost, plan questions | Plans start at Rs. 499/month plus pricing link |
| Refund | refund, return, cancel | 7-day refund policy, asks for order ID |
| Support | help, issue, login problems | Support contact details |
| Other | anything else | Acknowledgment, team replies within 24 hours (escalated) |

## Setup

### Prerequisites
- An n8n instance (cloud or self-hosted)
- OpenAI API key
- Telegram bot (create with [@BotFather](https://t.me/BotFather)) and your chat ID
- Google account for Google Sheets

### 1. Create the Google Sheet
Create a sheet with these headers in row 1:

```
Timestamp | Name | Email | Query | Category | Sentiment | Priority
```

### 2. Import the workflows
In n8n: **Workflows -> Import from File**, then import both files from the `workflows/` folder.

### 3. Add credentials
The exported files contain **no credentials**. In each workflow, open these nodes and select or create credentials:
- **OpenAI Chat Model**: OpenAI API key
- **Telegram nodes**: Telegram bot token
- **Google Sheets nodes**: Google OAuth2

### 4. Replace placeholders

| Placeholder | Where | Replace with |
|---|---|---|
| `YOUR_TELEGRAM_CHAT_ID` | All Telegram nodes | Your Telegram chat ID |
| `YOUR_GOOGLE_SHEET_ID` | All Google Sheets nodes | Sheet ID from the sheet URL (between `/d/` and `/edit`) |

### 5. Activate
Turn both workflows to **Active** and use the **Production URL** of the webhook.

## Testing

Send a test request (use the Test URL while building):

```bash
curl -X POST "YOUR_WEBHOOK_URL" \
  -H "Content-Type: application/json" \
  -d '{"name":"Anjali","email":"anjali@example.com","message":"How much is your premium plan?"}'
```

| Name | Message | Expected result |
|---|---|---|
| Anjali | How much is your premium plan? | Pricing, Normal, no escalation |
| Aditi | I want my money back, this is the worst service! | Refund, Negative, High, escalated |
| Sudha | I need help logging in | Support, Normal |
| Abha | Do you have an office in Delhi? | Other, escalated |
| Rehana | URGENT: my payment failed twice!! | Support, High, escalated |
| Anushka | Thank you, great product! | Other, escalated |

## Known Limitations and Ideas

- Customer replies are sent through Telegram to a single fixed chat ID. For real customers, replace `customer_telegram` with a **Send Email** node using `{{ $('Code in JavaScript').item.json.email }}`.
- The daily report compares dates in **UTC**. Adjust the date logic or the workflow timezone if your reports look a day off.
- The AI is prompted to return JSON only; the Code node falls back to a safe default (High priority) if parsing fails.
- Possible next steps: Slack alerts, a dashboard, multi-language replies, and storing a ticket ID.

## Security

- Never commit credentials, bot tokens, API keys, or real customer data.
- Exported workflows here have credential IDs, chat IDs, and sheet IDs removed.
- If you ever pushed a real token by mistake, revoke it and generate a new one.

## Tech Stack

n8n, OpenAI (GPT-4o-mini), Telegram Bot API, Google Sheets API, JavaScript

## Author

Abha
