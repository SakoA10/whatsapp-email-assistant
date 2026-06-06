# WhatsApp Email Assistant

Run your Gmail from WhatsApp. It sorts every incoming email, filters out the noise, and pings you only for mail that actually needs a reply. You answer in plain language, by text or voice note, and it drafts a clean, professional reply. You approve or tweak it in the chat, and once you approve, it replies in the same email conversation.

Built as a single [n8n](https://n8n.io) workflow.

## Flow

Gmail → filter → WhatsApp alert → your reply (text/voice) → draft → approve or revise → reply sent in the same thread

## Screenshots

**New email → WhatsApp alert**
![Email alert](screenshots/email-alert.png)

**Voice note → draft → CONFIRM → sent**
![Voice note to sent](screenshots/voice-confirm-send.png)

**Revise in plain language**

The draft it sends back first:
![Original draft](screenshots/revise-before.png)

After you say "make it shorter":
![Shorter draft](screenshots/revise-after.png)

**The reply lands in the Gmail thread**
![Reply in the Gmail thread](screenshots/gmail-in-thread.png)

**Several waiting? It asks which one**
![Choosing which email](screenshots/which-one.png)

**The workflow**
![n8n workflow](screenshots/n8n-canvas.png)

## What it does

- **Sorts and filters** — works out what each new email is (client, recruiter, lead, personal, finance…) and only alerts you for the ones worth a reply. Newsletters, promos, and no-reply blasts are skipped.
- **Alerts on WhatsApp** — sender, subject, priority, and a one-line summary.
- **Reply by text or voice** — voice notes are turned into text, and it reads messy spoken instructions (fillers, self-corrections) to work out what you actually mean.
- **Writes in your style** — short and professional, sent back for you to check.
- **Approve / revise / cancel / dismiss** — `CONFIRM` to send, or just say what to change and it rewrites it.
- **Replies in the same conversation** — the reply is sent from Gmail in the original thread.
- **Picks the right email** — several waiting? Swipe-reply the one you mean, reply with its number, or it asks which.
- **Won't email anyone but you** — it can't send to anyone else until you deliberately turn that off.

## How it works

One workflow, three sections (see the canvas above):

1. **Reads & filters your email** — new email in Gmail → pull out the key details → decide if it's worth a reply (when unsure, it shows you) → alert on WhatsApp and add it to the list.
2. **You reply from WhatsApp** — your text or voice → turn voice into text → work out which email you mean and what you want (common replies handled instantly, the rest go to the AI).
3. **Draft, Approve & Send** — write the reply → you check it → on `CONFIRM`, make sure it's only going to you → send from Gmail in the same thread → mark done. Ask for changes and it rewrites. Cancel or dismiss to drop it.

It keeps track of everything in two n8n Data Tables: `email_queue` and `reply_sessions`.

## Tech Stack

- **n8n** — runs the whole flow (single workflow, ~46 nodes)
- **Google Gemini** — sorting emails, writing drafts, voice-to-text, understanding your replies
- **WhatsApp Business Cloud API** — where you get alerts and reply
- **Gmail API** — reads incoming mail, sends replies in the same conversation

## Setup

Import `workflows/email_assistant.json`, then connect your own accounts:

1. **Credentials** — Google Gemini, WhatsApp Cloud API (sending), WhatsApp Trigger (receiving), Gmail OAuth.
2. **Data Tables** — create two and point each Data Table node at yours:
   - `email_queue`: email_id, thread_id, from_addr, subject, snippet, received_at, category, priority, summary, status, wa_message_id
   - `reply_sessions`: email_id, thread_id, recipient, subject, draft_text, state
3. **Your numbers** — set `recipientPhoneNumber` and `phoneNumberId` on the WhatsApp nodes (keep as `={{ '...' }}`).
4. **Your email** — set it in the Self-Only Gate (the only address it may send to).
5. **Your voice** — edit the Draft and Classify prompts with your name and tone.
6. **Public URL** — the WhatsApp trigger needs a public HTTPS address (a tunnel or hosted n8n).

The exported workflow holds credential references only, with no keys or tokens. Personal values are placeholders (`YOUR_EMAIL@example.com`, etc.).

## Notes

- **Revise by replying to the alert, not the draft.** Today you swipe-reply the original email alert. Replying to the draft message itself isn't picked up yet, since each email is tracked by its alert. Tracking the draft message is the next step.
- **Spam-filtered mail is invisible** — the Gmail trigger watches the inbox only.
- **Shortening can drift** — "make it shorter" occasionally rewords meaning, but you review before sending, so it's caught.
- **Single workflow** — kept as one for clarity. Heavy use would split the reading and replying into two.
