# estimate-follow-up-sequence
Daily n8n workflow that follows up on pending HVAC estimates on day 2, 5, and 10. Writes each email with OpenAI, sends it by Gmail, saves the stage in Supabase, and alerts on Telegram if a send fails.
