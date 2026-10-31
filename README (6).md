# n8n Global Error Handling Workflow

An n8n setup that detects failures in other workflows, classifies their severity, logs them to Google Sheets, alerts an admin on Slack, and produces a periodic summary and dashboard.

## Features

- **Error detection:** the Error Trigger catches failures from any workflow that points to it.
- **Structured metadata:** workflow name, failed node, error message, execution ID, execution URL and timestamp.
- **Severity levels:** errors are classified as Critical, Warning or Info by keyword rules.
- **Logging:** every error is appended to a Google Sheet.
- **Real-time alert:** a Slack message with a colour-coded severity and a link to the failed execution.
- **Handler safety net:** a separate fallback workflow alerts you if the handler itself fails.
- **Summary report:** a scheduled Slack summary of errors by severity and workflow.
- **Retry for transient errors:** "Retry On Fail" on external-service nodes (3 tries, 2-5 s wait).
- **Dashboard:** Google Sheets charts for errors per day, per workflow and per severity, plus a total.

## Architecture

```
Any monitored workflow ──(fails)──▶ Error_detection
                                      │
      Error Trigger → Extract Error Details → Add Severity → Slack alert → Google Sheets log
                                      │
                      (if the handler itself fails)
                                      ▼
                           Error handler Fallback → Slack "handler failed"

Error Summary Report:  Schedule → Get rows (Sheets) → Build summary (Code) → Slack
```

Slack runs **before** Sheets on purpose. If logging fails, the admin still gets the alert, and the failed handler run triggers the fallback.

## Repository contents

| File | Purpose |
|---|---|
| `workflows/01-error-detection.json` | Main handler (Error Trigger, severity, Slack, Sheets) |
| `workflows/02-error-handler-fallback.json` | Alerts when the handler fails |
| `workflows/03-error-summary-report.json` | Scheduled Slack summary |
| `workflows/04-test-failure.json` | Webhook workflow that throws a custom error for testing |

All credentials, IDs and webhook paths in these exports are placeholders.

## Setup

### 1. Google Sheet

Create a sheet named `error_log` with these headers in row 1 (spelling matters):

```
Workflow_name | Failed_node | Error_message | Execution_id | Status | Timestamp | Severity | Date | Retry_status
```

Optionally add a `Dashboard` tab (see below).

### 2. Credentials in n8n

Create two credentials: **Google Sheets OAuth2** and **Slack OAuth2**. The Slack app needs the `chat:write` scope, and the bot must be invited to the target channel.

### 3. Import the workflows

For each file in `workflows/`: n8n → **Create workflow** → **⋯ → Import from file**.

Then in each workflow:

1. Select your own Google Sheets and Slack credentials on the nodes.
2. Select your sheet and your Slack channel (the `YOUR_...` placeholders).
3. Under **Settings → Error Workflow**, wire the chain:
   - `Error handler Fallback` is set as the error workflow of **Error_detection**
   - **Error_detection** is set as the error workflow of every workflow you want to monitor (including the test workflow)
   - Optionally set Error_detection as the error workflow of the fallback too
4. Save and **Publish** all workflows.

If an import complains about the `REPLACE_WITH_...` error workflow ID, delete that setting before importing and select the workflow from the dropdown afterwards.

### 4. Schedule

`03-error-summary-report.json` is set to run **daily at 09:00**. Set the workflow timezone in **Settings** to match your sheet timestamps.

## Severity rules

Defined in the `Add Severity` Code node (checked on the lower-cased error message):

| Severity | Triggers on | Example |
|---|---|---|
| Critical | 401, 403, unauthorized, forbidden, permission, 500, cannot read, is not defined, database | `401 Unauthorized` |
| Warning | timeout, ETIMEDOUT, ECONNRESET, 429, 502-504, rate limit (and the default) | `429 Too Many Requests` |
| Info | required, validation, invalid input, 400, 404, not found | `Field 'topic' is required` |

Critical wins over Warning. For example, `Database connection timeout` is Critical.

## Testing

The Error Trigger fires only on **production** runs, not on manual "Execute" clicks.

1. Publish `04-test-failure.json` and copy its **Production URL**.
2. Send a POST with a JSON body, for example:

```bash
curl -X POST "https://<your-n8n-host>/webhook/test-failure" \
  -H "Content-Type: application/json" \
  -d '{"message": "401 Unauthorized"}'
```

| Body message | Expected severity |
|---|---|
| `Intentional test failure` | Warning |
| `401 Unauthorized` | Critical |
| `ETIMEDOUT connection timed out` | Warning |
| `Field 'topic' is required` | Info |
| `404 Not found` | Info |
| `Cannot read properties of undefined` | Critical |
| `Database connection timeout` | Critical |
| `429 Too Many Requests` | Warning |

Each request should produce a Slack alert, a new sheet row, and a successful `Error_detection` execution.

**Testing the fallback:** break the Sheets node (wrong document), keep its *On Error* setting at *Stop Workflow*, and trigger a failure. You should get the normal alert, a failed `Error_detection` run, and a message from the fallback workflow.

## Retry on transient errors

Set **Settings → Retry On Fail** (Max Tries 3, Wait 2000-5000 ms) on every node that calls an external service (HTTP, OpenAI, Sheets, Gmail, Slack). Skip Webhook, If, Code and Respond nodes. The error workflow runs only after all retries are exhausted.

## Dashboard (Google Sheets)

On a `Dashboard` tab:

```
A1: =QUERY(Sheet1!A:I,"select H, count(A) where H is not null group by H label count(A) 'Errors'",1)
E1: =QUERY(Sheet1!A:I,"select A, count(A) where A is not null group by A label count(A) 'Errors'",1)
I1: =QUERY(Sheet1!A:I,"select G, count(A) where G is not null group by G label count(A) 'Errors'",1)
M1: =COUNTA(Sheet1!A2:A)
```

Insert charts: column (errors per day), bar (by workflow), pie (by severity), scorecard (total). For richer filtering, connect the sheet to Looker Studio.

## Known limitations

- An error workflow cannot catch its own failures. That is why the separate fallback exists.
- Failures that are *handled* inside a workflow (for example a validation branch returning HTTP 400) do not count as errors, so they are not logged.
- Retry On Fail can duplicate a write in the rare case where the first attempt succeeded but timed out.
- The automatic re-run of failed executions through the n8n API was not implemented. It needs an API key.

## Security

- Never commit credentials, API keys or n8n instance IDs.
- Re-export workflows through the sanitizing step before committing changes (see `.gitignore`).
- Rotate any credential you accidentally commit.

## Tech stack

n8n Cloud, Google Sheets API, Slack API, JavaScript (Code nodes).
