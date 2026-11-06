# AI Intake Triage (n8n + Claude)

n8n workflows that watch a Gmail inbox, have Claude classify and prioritize each email, alert the team in Slack, and send a reply only after a person approves it in Slack.

They tackle the "too many emails, not enough time" problem small businesses have: sales leads, support requests, invoices and spam all land in the same inbox, and someone has to sort them by hand.

The repo has two versions:

| Version | Files | What it shows |
|---|---|---|
| **Production** | `workflow/intake-triage-*.json` (3 workflows) | Gmail trigger, routing, Slack alerts, human approval, sub-workflow, error handling |
| **Starter** | `workflow/ai-intake-triage.json` | The core idea with four sample emails and a manual trigger |

## Production version

```
New Email (Gmail, every minute)
  → Prep Email → Classify with Claude → Parse AI Reply
      ├→ Save to Table (every email)
      └→ Route
           ├ Spam           → Mark as read (no alert)
           ├ High priority  → Slack alert in #intake-alerts
           └ Needs reply    → Approve & Send Reply sub-workflow (one run per email)
                                → Slack message with Send reply / Don't send buttons
                                → Approved? → Gmail reply in the same thread → mark read

Any failure → Error Alerts workflow → Slack message with the workflow, step and error
```

### Design choices

- **A person approves every reply.** Claude drafts the reply, but nothing goes out until someone clicks **Send reply** in Slack. Requests expire after 24 hours without sending anything.
- **One approval run per email.** Slack's "send and wait" step handles one item per run, so a sub-workflow runs once for each email. Several emails arriving in the same minute each get their own approval.
- **Doesn't fail silently.** The Claude step retries up to 3 times, and an error workflow posts any failure to Slack.
- **Untrusted input stays data.** The email body is wrapped in markers that Claude is told to treat as data, not instructions. The parser also forces Claude's answer into the allowed categories and priorities.
- **Cost-aware.** It uses Claude Haiku, which is fast and cheap for classification.

### Test results

Real emails sent to a demo inbox:

| Email | Category | Priority | Result |
|---|---|---|---|
| Need a quote for automation work | sales lead | high | Slack alert. Reply approved and sent in the same thread |
| Can't reset my password | support | high | Slack alert and draft reply for approval |
| Question about my invoice | invoice | high | Slack alert. Reply declined, nothing sent |
| CONGRATULATIONS you won a $500 gift card!!! | spam | low | Marked as read, no alert |

<!-- Screenshots: add screenshots/*.png, then uncomment
![Workflow](screenshots/workflow.png)
![Slack approval](screenshots/slack-approval.png)
![Results table](screenshots/results-table.png)
-->

### Run it yourself

You need n8n (tested on 2.45.7), an Anthropic API key, a Gmail account with a Google Cloud OAuth client, and a Slack workspace with a bot (scopes `chat:write`, `chat:write.public`, `channels:read`).

1. Start n8n with `npx n8n` and open http://localhost:5678.
2. Create credentials for **Anthropic**, **Gmail OAuth2** and **Slack API**. For the Gmail OAuth client, the redirect URI is `http://localhost:5678/rest/oauth2-credential/callback`.
3. Create a Data table named `intake_triage` with these string columns: `received_at`, `from`, `subject`, `category`, `priority`, `summary`, `suggested_reply`.
4. Create a public Slack channel named `#intake-alerts`.
5. Import the workflows in this order: `intake-triage-error-alerts.json`, `intake-triage-approve-reply.json`, then `intake-triage-gmail-slack.json`. Pick your credentials in each node.
6. If n8n gave the workflows new IDs on import, point **Request Approval (per email)** at the Approve & Send Reply workflow. Then set the **Error workflow** under Settings in the main workflow and the sub-workflow.
7. Publish the error and approval workflows first, then the main one.

No API keys or tokens are stored in this repo. Each workflow only refers to its credentials by name.

## Starter version

`workflow/ai-intake-triage.json` runs four sample emails through Claude on a manual trigger and saves the results to the same table. It's the simplest way to see the classification step on its own.

## Ideas for extending it

- Add a webhook or Outlook trigger next to Gmail
- Route each category to its own follow-up flow, such as creating a CRM lead or opening a support ticket
- Give the approval message an "Edit reply" option that uses a Slack form
- Write to Google Sheets instead of an n8n Data table

## License

© 2026 John Pecillo. All rights reserved. Shared for portfolio review, not licensed for reuse. See [LICENSE](LICENSE).
