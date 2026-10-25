# Webhook to Google Sheets: Form Data Storage Workflow

An n8n automation that receives form submissions through a webhook, validates them, stores them as new rows in Google Sheets in real time, emails a confirmation to the submitter, and replies to the sender with a clear success or error message.

Built as Assignment 3 for the Tutedude automation course.

## Features

- Webhook trigger that accepts `POST` requests with a JSON body
- Email validation (required and correct format) before anything is stored
- Append-only storage in Google Sheets, so existing data is never overwritten
- Automatic timestamp for every submission
- `source` field that records which form sent the data
- Confirmation email sent to the submitter through Gmail
- Clear JSON responses: `200` on success and `400` on validation errors
- Two HTML forms that submit to the same webhook and show the result on the same page

## How it works

```
Webhook -> If (valid email?) -- true  --> Append row in sheet -> Gmail -> Respond to Webhook (200)
                             -- false --> Respond to Webhook1 (400)
```

1. A form or API client sends a `POST` request with the submission data.
2. The **Webhook** node receives it.
3. The **If** node checks that the email exists and matches a valid format.
4. If valid, **Google Sheets** appends a new row with a timestamp and source.
5. **Gmail** sends a confirmation email to the submitter.
6. **Respond to Webhook** returns a success reply to the sender.
7. If invalid, **Respond to Webhook1** returns a `400` error and nothing is saved.

## Nodes

| Node | Purpose |
|---|---|
| Webhook | Receives POST requests. Respond is set to "Using 'Respond to Webhook' Node". CORS is allowed so the HTML forms can call it. |
| If | Validates the email with a regular expression. True continues to storage, false goes to the error reply. |
| Append row in sheet | Operation "Append Row". Maps Name, Email, Phone, Message, Timestamp and Source to the sheet columns. |
| Gmail | Sends a confirmation email after the data is saved. Set to continue on error so a mail failure does not block the reply. |
| Respond to Webhook | Returns `200` with the success message. |
| Respond to Webhook1 | Returns `400` with the error message. |

## Google Sheet structure

Create a sheet with these headers in row 1:

| Name | Email | Phone | Message | Timestamp | Source |
|---|---|---|---|---|---|

## API reference

**Endpoint:** `POST https://<your-n8n-domain>/webhook/<your-webhook-path>`

**Request body**

```json
{
  "name": "Rahul Sharma",
  "email": "rahul@example.com",
  "phone": "9876543210",
  "message": "Hello, this is a test",
  "source": "postman"
}
```

**Success response** (`200`)

```json
{ "status": "success", "message": "Data saved successfully" }
```

**Error response** (`400`)

```json
{ "status": "error", "message": "A valid email is required" }
```

**Test with cURL**

```bash
curl -X POST "https://<your-n8n-domain>/webhook/<your-webhook-path>" \
  -H "Content-Type: application/json" \
  -d '{"name":"Rahul Sharma","email":"rahul@example.com","phone":"9876543210","message":"Hello","source":"curl"}'
```

## Setup

1. **Import the workflow.** In n8n, create a new workflow, open the menu, choose **Import from file**, and select `workflow/webhook-to-google-sheets.json`.
2. **Create the Google Sheet** with the headers listed above.
3. **Connect credentials.** Open the Google Sheets node and the Gmail node and select your own Google OAuth2 credentials. Choose your spreadsheet and sheet in the Google Sheets node.
4. **Set the webhook path** and, in the Webhook node options, set **Allowed Origins (CORS)** to `*`.
5. **Publish** the workflow and copy the **Production URL** from the Webhook node.
6. **Update the HTML forms.** Open each file in `forms/` and replace this line with your Production URL:
   ```js
   const WEBHOOK_URL = "PASTE_YOUR_PRODUCTION_WEBHOOK_URL_HERE";
   ```
7. Open a form in your browser and submit it, or send a request with Postman or cURL.

Use the **Test URL** (`/webhook-test/`) while building, and click **Listen for test event** before each request. The **Production URL** (`/webhook/`) works only after the workflow is published.

## HTML forms

| File | Description |
|---|---|
| `forms/contact-form.html` | Basic form that posts the data and shows the reply message under the button. |
| `forms/contact-form-success.html` | Replaces the form with a "Submitted successfully" message on the same page. |

Each form includes a hidden `source` field (`form1` and `form2`). Both forms use the same webhook URL, and each submission runs as a separate execution.

## Testing

- Valid submission returns `200` and adds a new row with a timestamp and source
- Several submissions add separate rows without overwriting
- Missing email returns `400` and adds no row
- Invalid email format returns `400` and adds no row
- Confirmation email arrives after a valid submission
- Both HTML forms work against the Production URL

## Screenshots

| | |
|---|---|
| Workflow canvas | ![Workflow canvas](screenshots/01-workflow-canvas.png) |
| Successful execution | ![Execution](screenshots/02-execution-success.png) |
| Google Sheet with rows | ![Google Sheet](screenshots/03-google-sheet.png) |
| Postman success | ![Postman success](screenshots/04-postman-success.png) |
| Postman error | ![Postman error](screenshots/05-postman-error.png) |
| HTML form success | ![Form success](screenshots/06-form-success.png) |
| Confirmation email | ![Email](screenshots/07-confirmation-email.png) |

Rename the files in `screenshots/` to match these names, or edit the paths above.

## Project structure

```
.
├── README.md
├── .gitignore
├── workflow/
│   └── webhook-to-google-sheets.json
├── forms/
│   ├── contact-form.html
│   └── contact-form-success.html
├── screenshots/
│   └── (workflow, execution, sheet, postman, form and email images)
└── docs/
    └── explanation.pdf
```

## Security notes

- The webhook URL is public, so anyone who has it can send data to it. For a real project, add authentication (for example a secret header) in the Webhook node.
- No credentials are stored in this repository. Google credentials are created inside your own n8n account.
- Replace the webhook URL, spreadsheet ID and any personal email addresses before publishing your files.

## Tech stack

n8n, Google Sheets API, Gmail API, HTML, CSS, JavaScript (fetch), Postman

## Author

**Your Name**
GitHub: [@your-username](https://github.com/your-username)
