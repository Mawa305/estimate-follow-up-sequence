# Estimate Follow-up Sequence

A daily n8n workflow for an HVAC business that follows up on pending estimates. It runs every morning, finds estimates that have not been accepted, and sends a follow-up email on day 2, day 5, and day 10. The email text is written by OpenAI. Each stage is sent once, and failures go to Telegram.

## Architecture

```
Schedule Trigger (daily, 09:00)
   → Get many rows (Supabase: estimates, status = pending)
   → Code in JavaScript (days since sent, due stage: 2, 5, or 10)
   → Message a model (OpenAI writes the email text)
   → Edit Fields (keeps follow_up_message)
   → Send a message (Gmail)
        ├── success → Update a row (Supabase: last_follow_up_stage = due stage)
        └── error   → Send a text message (Telegram)
```

## Features

* Runs every day at 09:00
* Sends at most one follow-up per estimate per run, using the most advanced stage that is due
* Day 2 is a gentle check-in, day 5 points out timing and value, and day 10 is a final reminder before the estimate expires
* The stage is saved after each send, so the same follow-up is not sent twice
* The email subject changes for the final reminder
* Failed sends are reported to Telegram with the customer name, email, and estimate ID

## Tech Stack

* n8n 2.x (workflow automation)
* Supabase (estimates table)
* OpenAI (follow-up text)
* Gmail API (customer emails)
* Telegram Bot API (failure alerts)

## Setup

1. Create a Supabase table named `estimates` with these columns:

```
id | customer_name | email | estimate_amount | sent_date | status | last_follow_up_stage
```

   Use `sent_date` in a format JavaScript can read, such as `2026-11-01`. Set `status` to `pending` for open estimates. Set `status` to something else (for example `accepted`) once the customer accepts or the estimate closes. Leave `last_follow_up_stage` empty or `0` for new estimates.

2. Import `estimate_follow_up_sequence.json` into n8n.
3. Add credentials: Supabase, OpenAI, Gmail OAuth2, and Telegram. Assign them to the matching nodes.
4. Replace `YOUR_TELEGRAM_CHAT_ID` in the Send a text message node with your chat ID.
5. Test with one estimate whose `sent_date` is 2 days ago. Check the email, then `last_follow_up_stage`, which should be `2`.
6. Activate the workflow when the test works.

## Follow-up Stages

| Days since sent | Stage | Email type |
|---|---|---|
| 2 to 4 | 2 | Gentle check-in |
| 5 to 9 | 5 | Value and timing reminder |
| 10 or more | 10 | Final reminder (subject changes) |

Only the most advanced stage that is due is sent on each run, so a late estimate jumps straight to the right email. A stage that is already recorded in `last_follow_up_stage` is skipped.

## Sample Row

```
customer_name:   Ali Raza
email:           ali.test@example.com
estimate_amount: 450
sent_date:       2026-11-01
status:          pending
```

If today is 4 November 2026, this estimate is 3 days old and gets the day 2 follow-up.

## Project Notes

* Built as part of my AI Automation portfolio.
* The stage, the timing, and the send decision are set in code. The model only writes the email text.
* Only the OpenAI message node uses `gpt-4o-mini`. Change its model in that node if you want another model.
