# AI Intake Triage (n8n + Claude)

An n8n workflow that reads incoming messages, has Claude classify and prioritize each one, drafts a first reply, and saves the results to a table.

It's a starter for the "too many emails, not enough time" problem small businesses have: sales leads, support requests, invoices and spam all land in the same inbox, and someone has to sort them by hand.

## What it does

| Step | Node | What happens |
|---|---|---|
| 1 | **When clicking Test workflow** | Manual trigger, so the demo can be run on demand |
| 2 | **Classify & Summarize** (Code) | Supplies four sample emails: a sales lead, a support request, an invoice and spam |
| 3 | **Message a model** (Anthropic) | Sends each email to Claude and asks for JSON only: `category`, `priority`, `summary`, `suggested_reply` |
| 4 | **Parse AI Reply** (Code) | Finds Claude's JSON in the response, parses it (falls back to `other` if it can't) and merges it with the original email fields |
| 5 | **Save to Table** (Data table) | Inserts one row per email into an n8n Data table named `intake_triage` |

### Result

All four sample emails come back correctly categorized, with a priority, a one-sentence summary and a draft reply (spam gets no reply).

| from | category | priority |
|---|---|---|
| maria@acmeco.com | sales lead | high |
| dave@randomuser.net | support | high |
| billing@vendorco.com | invoice | medium |
| promo@cheapdeals.biz | spam | low |

<!-- Screenshots: add screenshots/workflow.png and screenshots/results-table.png, then uncomment
![Workflow](screenshots/workflow.png)
![Results table](screenshots/results-table.png)
-->

## Run it yourself

1. Start n8n (tested on n8n 2.45.7): `npx n8n`, then open http://localhost:5678.
2. Create a Data table named `intake_triage` with these string columns: `received_at`, `from`, `subject`, `category`, `priority`, `summary`, `suggested_reply`.
3. Import `workflow/ai-intake-triage.json` (Workflows → Import from File).
4. Open **Message a model** and select or create an Anthropic credential with your own API key.
5. Open **Save to Table** and select your `intake_triage` table.
6. Click **Test workflow**.

No API keys or credentials are stored in this repo. The workflow only refers to a credential by name.

## Notes

- The model is set to `claude-opus-5-5`. For high-volume triage, a smaller model such as Claude Haiku is much cheaper and is usually accurate enough.
- The parser handles responses that include a thinking block alongside the text.

## Ideas for extending it

- Replace the sample data with a Gmail, Outlook or webhook trigger
- Send a Slack alert for high-priority messages
- Write to Google Sheets instead of an n8n Data table
- Route each category to a different follow-up flow
