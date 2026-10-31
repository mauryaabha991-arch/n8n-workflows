# n8n Workflows

A collection of my n8n automation workflows. Each workflow has its own folder with the JSON, a README, and screenshots.

| Workflow | Description |
|---|---|
| [RSS_News](RSS_News) | Emails the top 3 headlines every morning from BBC News and The Verge, with duplicate protection via Google Sheets |
| [Weather_alert](Weather_alert) | Emails a rain alert every morning at 7 AM for multiple cities using the OpenWeatherMap 24-hour forecast, with error alerts |
| [Webhook_toGoogleSheet](Webhook_toGoogleSheet) | Receives form data through a webhook, validates the email, saves it to Google Sheets with a timestamp and source, and sends a confirmation email |
| [Lead_Classification_workflow](Lead_Classification_workflow) | Receives leads through a webhook, validates and scores them (1-10), uses Gemini AI to read buying intent, classifies each as Hot, Warm, or Cold, saves to Google Sheets, and emails an alert for every Hot lead |
| [Multi_platform_notifier](Multi_platform_notifier) | Receives a notification through a webhook (subject, message, priority), routes it by priority (High, Medium or Low) to Gmail, Slack and Telegram in parallel, retries failed deliveries, and logs every attempt with status and timestamp to Google Sheets |
| [Price_drop_workflow](Price_drop_alert_workflow) | Checks product prices every 6 hours for multiple products, sends a Telegram alert when a price drops below its threshold, tracks price trend (rising/falling/stable), applies a 24-hour cooldown to avoid repeat alerts, and logs every check and API error to Google Sheets |
| [Customer_query_reply](Customer_query_reply) | Receives customer queries through a webhook, uses OpenAI (GPT-4o-mini) to detect the category (Pricing, Refund, Support, Other) and sentiment, writes a personalized reply, alerts the team on Telegram for urgent, negative or unrecognized queries, logs every query to Google Sheets, and sends a daily summary report |
| [Ai_content_generator](Ai_content_generator) | Receives a topic and content type through a webhook, validates the input and checks Google Sheets for duplicates (returning stored content if found), writes the content with an OpenAI AI Agent in the chosen tone, length and language, saves it to Google Sheets, sends it for Gmail approve/reject, posts approved content to Slack, and logs API errors |
| [Error_Detection](Error_Detection) | Catches failures from any monitored workflow with the Error Trigger, extracts the workflow name, failed node, error message and execution ID, classifies severity (Critical, Warning or Info), sends a Slack alert with a link to the failed execution, and logs every error to Google Sheets. A separate fallback workflow alerts you if the handler itself fails, and a scheduled workflow posts a daily error summary to Slack |

## Tools used

n8n, Google Sheets, Gmail, OpenWeatherMap API, RSS feeds, Webhooks,Google Gemini,Slack,Telegram,DummyJSON API,OpenAI (GPT-4o-mini)

## How to use a workflow

1. Open the workflow's folder and import the JSON file into n8n (**Import from file**).
2. Connect your own credentials (Google, Gmail, API keys) in the nodes that need them.
3. Follow the folder's README for the remaining setup steps.

No credentials are stored in this repository.
