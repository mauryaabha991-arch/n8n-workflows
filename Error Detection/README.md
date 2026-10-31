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
- **Retry for transient errors:** Retry On Fail on external-service nodes (3 tries, 2-5 s wait).
- **Dashboard:** Google Sheets charts for errors per day, per workflow and per severity, plus a total.

## Architecture

```
Any monitored workflow ──(fails)──▶ Error_detection

  Error Trigger → Extract Error Details → Add Severity → Slack alert → Google Sheets log
                                  │
                  (if the handler itself fails)
                                  ▼
                       Error handler Fallback → Slack "handler failed"

Error Summary Report:  Schedule → Get rows (Sheets) → Build summary (Code) → Slack
```

Slack runs **before** Sheets on purpose. If logging fails, the admin still gets the alert, and the failed handler run triggers the fallback.

## Folder structure

```
Error Detection/
├── README.md
├── Workflows/
│   ├── error-detection.json          Main handler
│   ├── error-handler-fallback.json   Alerts when the handler fails
│   ├── error-summary-report.json     Scheduled Slack summary
│   └── test-failure.json             Webhook workflow that throws a custom error
└── Workflow Screenshots/
    ├── Screenshot Error Detection.png
    ├── Screenshot Error Handler Fallback.png
    ├── Screenshot Error Summary Report.png
    └── Screenshot Test Failure.png
```

All credentials, IDs and webhook paths in the exported JSON files are placeholders.

## Screenshots

**Error Detection (main handler)**

![Error Detection](<Workflow Screenshots/Screenshot Error Detection.png>)

**Error handler Fallback**

![Fallback](<Workflow Screenshots/Screenshot Error Handler Fallback.png>)

**Error Summary Report**

![Summary](<Workflow Screenshots/Screenshot Error Summary Report.png>)

**Test Failure workflow**

![Test Failure](<Workflow Screenshots/Screenshot Test Failure.png>)

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

For each file in `Workflows/`: n8n → **Create workflow** → **⋯ → Import from file**.

Then in each workflow:

1. Select your own Google Sheets and Slack credentials on the nodes.
2. Select your sheet and your Slack channel (replace the `YOUR_...` placeholders).
3. Under **Settings → Error Workflow**, wire the chain:
   - `Error handler Fallback` is the error workflow of **Error_detection**
   - **Error_detection** is the error workflow of every workflow you want to monitor, including the test workflow
4. Save and **Publish** all four workflows.

If an import complains about a `REPLACE_WITH_...` error workflow ID, remove that setting before importing and pick the workflow from the dropdown afterwards.

### 4. Schedule

`error-summary-report.json` runs **daily at 09:00**. Set the workflow timezone in **Settings** to match your sheet timestamps.

## Severity rules

Defined in the `Add Severity` Code node (checked on the lower-cased error message):

| Severity | Triggers on | Example |
|---|---|---|
| Critical | 401, 403, unauthorized, forbidden, permission, 500, cannot read, is not defined, database | `401 Unauthorized` |
| Warning | timeout, ETIMEDOUT, ECONNRESET, 429, 502-504, rate limit (and the default) | `429 Too Many Requests` |
| Info | required, validation, invalid input, 400, 404, not found | `Field 'topic' is required` |

Critical wins over Warning. For example, `Database connection timeout` is Critical.

## Testing with Postman

The Error Trigger fires only on **production** runs, not on manual "Execute" clicks.

1. Import and **Publish** `test-failure.json`, and copy the **Production URL** from its Webhook node (the path with `/webhook/`, not `/webhook-test/`).
2. In Postman, create a request:
   - **Method:** `POST`
   - **URL:** `https://<your-n8n-host>/webhook/test-failure`
   - **Body:** `raw` → `JSON`
3. Send each body below, waiting about 10 seconds between requests.

| # | Postman body | Expected severity |
|---|---|---|
| 1 | `{"message": "Intentional test failure"}` | Warning |
| 2 | `{"message": "401 Unauthorized"}` | Critical |
| 3 | `{"message": "ETIMEDOUT connection timed out"}` | Warning |
| 4 | `{"message": "Field 'topic' is required"}` | Info |
| 5 | `{"message": "404 Not found"}` | Info |
| 6 | `{"message": "Cannot read properties of undefined (reading 'name')"}` | Critical |
| 7 | `{"message": "Database connection timeout"}` | Critical |
| 8 | `{"message": "429 Too Many Requests"}` | Warning |

For each request, confirm:

- A **Slack alert** arrives with the right colour (🔴 Critical, 🟠 Warning, 🔵 Info), the workflow name, the failed node, the error message and an execution link.
- A **new row** appears in the sheet with Severity, Date and Timestamp filled in.
- A new `Error_detection` execution appears, marked Succeeded.

Postman will show an error response for these calls, which is expected because the test workflow throws on purpose.

**Testing the fallback:** point the Sheets node at the wrong document, keep its *On Error* setting at *Stop Workflow*, and send request #2. You should get the normal Slack alert, a failed `Error_detection` run, and a message from the fallback workflow. Restore the Sheets node afterwards.

**Testing the summary:** open `Error Summary Report` and click **Execute workflow**. The Slack message should show the total, counts by severity and workflow, and the latest five errors.

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

Insert charts: column (errors per day), bar (by workflow), pie (by severity), scorecard (total).

## Known limitations

- An error workflow cannot catch its own failures, which is why the separate fallback exists.
- Failures handled inside a workflow (for example a validation branch returning HTTP 400) are not errors, so they are not logged.
- Retry On Fail can duplicate a write in the rare case where the first attempt succeeded but timed out.
- Automatic re-run of failed executions through the n8n API was not implemented, since it needs an API key.

## Tech stack

n8n Cloud, Google Sheets API, Slack API, JavaScript (Code nodes), Postman (testing).

## Security

No credentials are stored in this repository. Never commit API keys, OAuth tokens or n8n instance IDs, and rotate anything that was committed by mistake.
