# n8n Workflows

A collection of my n8n automation workflows. Each workflow has its own folder with the JSON, a README, and screenshots.

| Workflow | Description |
|---|---|
| [RSS_News](RSS_News) | Emails the top 3 headlines every morning from BBC News and The Verge, with duplicate protection via Google Sheets |
| [Weather_alert](Weather_alert) | Emails a rain alert every morning at 7 AM for multiple cities using the OpenWeatherMap 24-hour forecast, with error alerts |
| [Webhook_toGoogleSheet](Webhook_toGoogleSheet) | Receives form data through a webhook, validates the email, saves it to Google Sheets with a timestamp and source, and sends a confirmation email |

## Tools used

n8n, Google Sheets, Gmail, OpenWeatherMap API, RSS feeds, Webhooks

## How to use a workflow

1. Open the workflow's folder and import the JSON file into n8n (**Import from file**).
2. Connect your own credentials (Google, Gmail, API keys) in the nodes that need them.
3. Follow the folder's README for the remaining setup steps.

No credentials are stored in this repository.
