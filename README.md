# Customer Support Ticket Bot (n8n + Telegram + Local Gemma 4)

An n8n workflow that receives Telegram messages, classifies them with a locally hosted Gemma 4 model (Ollama), routes by the AI-assigned category, logs every ticket to Google Sheets and replies automatically.
<img width="1534" height="769" alt="image" src="https://github.com/user-attachments/assets/2a33bdda-8dcf-4b2b-bbf2-fee1100bf284" />


## How it works

1. **Telegram Trigger** receives the message (webhook via ngrok).
2. **Input checks:** empty/non-text messages get a polite reply; `/history` returns the user's last 5 tickets; rate limiter allows 5 messages/minute per user.
3. **AI classification:** HTTP Request to Ollama (`POST /api/chat`, model `gemma4:e4b`, `format: json`). The system prompt restricts output to `{"category": "billing|technical|general", "language": "az|en|ru"}`.
4. **Validation:** a Code node parses the JSON and marks anything outside the allowed labels as `unknown`.
5. **Retry:** `unknown` triggers one retry with a stricter prompt.
6. **Switch** routes on the AI category: billing, technical, general, plus a fallback output.
7. **Logging:** every ticket is appended to Google Sheets (timestamp, user, category, message, chatId).
8. **Replies:** a distinct templated reply per category, in the user's language (Azerbaijani / English / Russian).
9. **External API:** the general branch calls `open.er-api.com` and adds the live USD/AZN rate to the reply.
10. **Admin notification:** technical tickets also notify the admin via Telegram.

## Error handling
| Case | Behaviour |
|---|---|
| Empty / non-text message | `Reply Empty` asks for text |
| Ollama not running / timeout | Error output of the Gemma node -> `Reply AI Down` |
| Malformed JSON / unknown category | Code node sets `unknown` -> retry -> fallback reply |
| Exchange-rate API failure | Node continues; reply is sent without the rate |
| Too many messages | `Reply Rate Limited` |

## Bonus features
Admin notification, language detection (az/en/ru), retry with stricter prompt, rate limiter, `/history` command.

## Setup
- n8n self-hosted (`npx n8n`) with `WEBHOOK_URL` set to the ngrok HTTPS domain
- Ollama with `ollama pull gemma4:e4b`, called at `http://127.0.0.1:11434`
- ngrok static domain forwarding to port 5678
- Telegram bot from @BotFather (credential created manually in n8n)
- Google Sheets (OAuth credential), columns: `timestamp, user, category, message, chatId`
- No credentials are included in `workflow.json`

## Logging store
Google Sheets.

## Known limitations
- The rate limiter counts tickets from Sheets timestamps, which are written after the AI reply, so messages sent faster than the bot replies can exceed the limit.
