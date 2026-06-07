# WhatsApp Email Assistant

Run your Gmail from WhatsApp. It sorts every incoming email, filters out the noise, and pings you only for mail that actually needs a reply. You answer in plain language, by text or voice note, and it drafts a clean, professional reply. You approve or change it in the chat, and once you approve, it replies in the same email conversation.

Built as a single [n8n](https://n8n.io) workflow.

## Flow

Gmail → filter → WhatsApp alert → your reply (text/voice) → draft → approve or change → reply sent in the same conversation

## Screenshots

**New email → WhatsApp alert**
![Email alert](screenshots/email-alert.png)

**Voice note → draft → CONFIRM → sent**
![Voice note to sent](screenshots/voice-confirm-send.png)

**The reply lands in the Gmail conversation**
![Reply in the Gmail conversation](screenshots/gmail-in-thread.png)

**Several waiting? It asks which one**
![Choosing which email](screenshots/which-one.png)

**Change it in plain language**

The draft it sends back first:
![Original draft](screenshots/revise-before.png)

After you say "make it shorter":
![Shorter draft](screenshots/revise-after.png)

**The workflow**
![n8n workflow](screenshots/n8n-canvas.png)

## What it does

- **Sorts and filters** — works out what each new email is (client, recruiter, lead, personal, finance…) and only alerts you for the ones worth a reply. Newsletters, promos, and no-reply blasts are skipped.
- **Alerts on WhatsApp** — sender, subject, priority, and a one-line summary.
- **Reply by text or voice** — voice notes are turned into text, and it reads messy spoken instructions (fillers, self-corrections) to work out what you actually mean.
- **Writes in your style** — short and professional, sent back for you to check.
- **Approve / change / cancel / dismiss** — `CONFIRM` to send, or just say what to change and it rewrites it.
- **Replies in the same conversation** — the reply is sent from Gmail as part of the original email, not a new message.
- **Picks the right email** — several waiting? Swipe-reply the one you mean, reply with its number, or it asks which.
- **Safe test mode** — out of the box it only ever emails you, so you can test without accidentally messaging a real person. You lift this gate deliberately when you're ready to go live.

## How it works

One workflow, three sections (see the canvas above):

1. **Reads & filters your email** — new email in Gmail → pull out the key details → decide if it's worth a reply (borderline ones lean toward yes) → alert you on WhatsApp.
2. **You reply from WhatsApp** — your text or voice → turn voice into text → work out what you're asking for.
3. **Draft, Approve & Send** — write the reply → you check it → on `CONFIRM`, make sure it's only going to you → send from Gmail in the same conversation → mark done. Ask for changes and it rewrites. Cancel or dismiss to drop it.

Under the hood, it uses two n8n Data Tables: `email_queue` and `reply_sessions`.

## Tech Stack

- **n8n** — runs the whole flow (single workflow, ~47 nodes)
- **Google Gemini** — sorting emails, writing drafts, voice-to-text, understanding your replies
- **WhatsApp Business Cloud API** — where you get alerts and reply
- **Gmail API** — reads incoming mail, sends replies in the same conversation

## Setup

**Requirements** — a running n8n, self-hosted or Cloud, on a recent release. The workflow uses n8n **Data Tables** and the **Google Gemini** node, which both need a current version, so update n8n if yours is old. You'll also need accounts for Google Gemini, WhatsApp Business Cloud API, and Gmail.

**Import** — in n8n, go to **Workflows → Import from File** and import both files from `workflows/`: `email_assistant.json` (the assistant) and `setup_tables.json` (a one-time setup helper). Then work through the steps below.

1. **Create the Data Tables** — open `setup_tables.json` and click **Execute workflow** once. It creates the two tables the assistant needs, reuses them if they already exist, and is safe to re-run. The main workflow finds them by name, so there's nothing to re-point.
   - `email_queue`: email_id, thread_id, from_addr, subject, snippet, received_at, category, priority, summary, status, wa_message_id
   - `reply_sessions`: email_id, thread_id, recipient, subject, draft_text, state
2. **Connect your credentials** — the import brings the nodes but not the keys, so credentialed nodes show **"credential not found"** until you create your own and select them. You need four:
   - **Google Gemini** — an API key from Google AI Studio.
   - **WhatsApp Cloud API (sending)** — a permanent access token from your Meta app, plus your phone number ID.
   - **WhatsApp Trigger (receiving)** — the same Meta app, for inbound messages.
   - **Gmail OAuth** — a Google Cloud OAuth client with the Gmail API enabled.
3. **Set your WhatsApp numbers** — on every WhatsApp node that has them, 11 in all (Alert on WhatsApp, Send Draft for Review, Sent Confirm, and so on), set `recipientPhoneNumber` (your own number, where alerts arrive) and `phoneNumberId` (from your Meta WhatsApp app). Keep the `={{ '...' }}` format.
4. **Set your email, in safe test mode** — put your own address in the **Self-Only Gate** node. While it's set, the assistant **only ever emails you**, so you can test without messaging anyone real. When you're ready to go live and reply to the actual senders, change or disable that gate.
5. **Make it sound like you** — replace `YOUR_NAME` everywhere it appears, across four nodes: **Classify Email**, **Draft Reply**, **Read Intent**, and **Extract Draft**. In **Extract Draft** the name is part of a sign-off rule (`Best regards,` then your name), so use the exact same name you put in the Draft prompt or the sign-off won't format right. Adjust the Draft and Classify wording to your tone while you're in there.
6. **Give the trigger a public URL** — WhatsApp has to reach n8n over public HTTPS. Set n8n's `WEBHOOK_URL` to your public address, expose it with a tunnel (cloudflared or ngrok) or use n8n Cloud, then add that URL as the callback, with your verify token, in your Meta app's WhatsApp webhook settings.
7. **Activate the workflow** — switch the **Email Assistant** workflow to **Active** so the Gmail trigger starts checking mail and the WhatsApp webhook registers. Nothing runs until it's active.

The exported workflow holds credential references only, with no keys or tokens, and personal values are placeholders (`YOUR_EMAIL@example.com`, etc.). The Gemini nodes use specific models (`gemini-3.1-flash-lite` and `gemini-3.5-flash`). If your account doesn't offer those exact names, open the Gemini nodes and pick an available one.

## Notes

- **Swipe the draft or the alert** — to change or confirm a reply, swipe-reply either the draft message or the original email alert. Both point to the same email, so either works.
- **Spam-filtered mail is invisible** — the Gmail trigger watches the inbox only.
- **Edits keep your facts** — "make it shorter" and other changes revise the existing draft and preserve the details (dates, prices, names), and you review before anything sends.
- **Single workflow** — kept as one for clarity. Heavy use would split the reading and replying into two.
