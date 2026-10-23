# Project Request Orchestrator

A multi-agent proposal workflow built in n8n. A classifier agent labels each client request by service type, a specialist agent drafts a proposal, and a QA agent reviews it. Proposals with a budget or estimated price above 1000, or ones QA rejects, go to a human on Telegram before the client gets an email.

## Architecture

```
Webhook → Validate Input → Input Valid?
   → Classify Agent → Parse Classification → Build Specialist Prompt
   → Specialist Agent → Parse Proposal → Build QA Prompt → QA Agent → Parse QA
   → Ready for Review?
        ├── No (revise, max 2) → Build Specialist Prompt
        └── Yes → Needs Approval?
              ├── Yes → Telegram Approval → Read Approval → Log Request
              └── No  → Auto Approve → Log Request
   → Log Request → Send Client Email → Respond Success
```

## Features

* Classifier agent routes requests to `seo`, `content`, `ads`, `web`, or `general`
* Specialist agent drafts a proposal (title, scope, timeline, estimated price, summary) with a service-specific system prompt
* QA agent reviews the draft and sends it back for revision, up to two rounds
* Human-in-the-loop approval on Telegram when the budget or estimated price is above 1000, or QA rejects the draft
* Every outcome is logged to Google Sheets with status and approval source
* Client receives an email with the proposal summary or a "under review" notice
* Webhook returns a JSON response for invalid input (400) and for processed requests
* Client text is wrapped in tags and treated as data, which guards against prompt injection
* Separate error handler workflow sends failures to Telegram

## Tech Stack

* n8n 2.x (workflow automation)
* OpenAI `gpt-5-mini` (classifier, specialist, and QA agents)
* Telegram Bot API (approval requests and error alerts)
* Google Sheets (request log)
* Gmail API (client emails)

## Setup

1. Import `error_handler_workflow.json` into n8n, then import `project_request_orchestrator.json`.
2. Add credentials: OpenAI, Telegram, Google Sheets OAuth2, and Gmail OAuth2.
3. Replace `YOUR_TELEGRAM_CHAT_ID` with your own chat ID and `YOUR_SHEET_ID` with your Google Sheet ID.
4. Create a Google Sheet with a tab named `Requests` and these headers in row 1:

```
Timestamp | RequestID | Name | Email | ServiceType | Budget | EstimatedPrice | Status | ApprovedBy | Revisions
```

5. In the main workflow settings, set the Error Workflow to the error handler.
6. Activate the workflow and send a POST request to the webhook.

## Sample Request

```
{
  "name": "Ali Raza",
  "email": "ali.test@example.com",
  "request": "Need an SEO audit and keyword plan for my dental clinic website, including on-page fixes and a 3-month content outline.",
  "budget": 1500
}
```

With a budget of 1500, the request goes to Telegram for approval. Use a budget of 500 to see the auto-approve path.

## Project Notes

* Built as part of my AI Automation portfolio.
* The approval threshold (1000) is set in the Parse QA node and can be changed there.
* Only `gpt-5-mini` is used. Temperature and response format options are left at their defaults.
