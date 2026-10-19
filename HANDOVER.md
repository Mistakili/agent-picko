# Agent Picko handover

This is the manual for the next owner. Agent Picko is a voice-first CRM for real-estate agents. A person talks to the app, and the app saves leads and follow-ups for that signed-in account.

The live demo the seller has been using is a separate deployment. This repository does not include that deployment’s keys, database, or accounts. You bring your own.

## What works

- Email and password accounts. Each account has its own leads, follow-ups, and handoff notes.
- Browser voice. The microphone streams audio to AssemblyAI. Picko greets the caller, and can create a lead, look one up, update details the caller actually gave, and save a follow-up.
- A workspace with Home, Leads, and Follow-ups. Home is the voice screen. Leads and Follow-ups show the saved book.
- A saved request when the caller asks for a person. Picko tells them someone will call back.

## What it does not do

- It does not place or answer a phone call. Twilio is not connected. The handoff route stores the request and, when Twilio credentials are absent, returns `pending_setup`.
- It does not reset a forgotten password, and it does not offer Google sign-in.
- It does not include AssemblyAI credits. The key on the seller’s account stays there.
- It does not share data with the separate Client Brain product. Use a database that belongs to this app.

## Voice cost

AssemblyAI Voice Agent bills the whole time the voice socket stays open, including silence. The published rate used for this product is about **$0.075 per minute** ($4.50 per hour). A conversation costs more than a short voice note.

The browser sends `{ "type": "session.end" }` before it closes the socket, including when the tab is closing. Closing without that message leaves AssemblyAI a billable resume window of about 30 seconds.

Use headphones. The microphone will hear Picko’s reply if the speakers are on.

## First hour

You need Node 22.13 or newer, an AssemblyAI API key, and the Vercel CLI.

1. Copy `.env.example` to `.env.local`.
2. Set `ASSEMBLYAI_API_KEY` to your own key.
3. Set `CLIENTBRAIN_TOOL_SECRET` to a new random string of at least 16 characters. Generate a new one. Do not reuse a value from anyone else.
4. For a local run, set `APP_ORIGIN` and `ALLOWED_ORIGIN` to `http://localhost:3000`.
5. Leave `DATABASE_URL` empty for the first run. Accounts are stored in `data/clientbrain-plus.db`.
6. Run `npm install`, then `vercel dev`.
7. Open the URL Vercel prints. Create an account. Put on headphones. Tap the microphone and say a new buyer’s name, what they want, where, and a budget. Open Leads and confirm that person is there.

`npm run dev` only serves the page. The account and voice routes live in `api/` and run under `vercel dev`.

`npm test` runs the name-matching, voice-config, and book tests.

## Deploy

Create a new Vercel project from this repo. Do not point it at anyone else’s database.

Set these production variables:

| Name | Purpose |
| --- | --- |
| `ASSEMBLYAI_API_KEY` | Your AssemblyAI key. Server only. |
| `CLIENTBRAIN_TOOL_SECRET` | At least 16 characters. CRM and handoff routes refuse to run without it. |
| `DATABASE_URL` | Postgres connection string for this app. |
| `DATABASE_URL_UNPOOLED` | Direct Postgres URL. The app prefers this over `DATABASE_URL` so `node-pg` does not fight a pooled connection. |
| `APP_ORIGIN` | The public URL, with no trailing slash. Optional. When it is empty, tool URLs use the deployment host. |
| `ALLOWED_ORIGIN` | Extra browser origins allowed to request a voice token, comma-separated. Localhost is always allowed. |

`CLIENTBRAIN_DB_PATH` can override the local SQLite file. Leave it unset unless you need a different path.

Pushing `main` deploys production when the Vercel project is connected to GitHub. After the first deploy, sign up on the public URL and repeat the spoken-lead check.

## How voice is wired

The browser and the phone agent share one prompt, greeting, voice (`ivy`), and tool list in `api/_lib/voice.js`.

Browser path:

1. The signed-in page calls `GET /api/voice-config` and `GET /api/voice-token`.
2. The token route asks AssemblyAI for a short-lived token. The permanent key never reaches the browser.
3. The page opens `wss://agents.assemblyai.com/v1/ws` and sends `session.update` with the config from this server.
4. Microphone audio is PCM16 at 24 kHz from `public/pcm-processor.js`.
5. When Picko calls a tool, the page posts it to `/api/crm` or `/api/transfer` with the session cookie.

Phone path:

1. `ensureAgent` in `api/_lib/agents.js` creates or updates an AssemblyAI agent for that account.
2. Tool URLs are built from `APP_ORIGIN`, or from this deployment’s host. A caller’s `Origin` header cannot choose a different host.
3. The secret travels in the header `X-ClientBrain-Tool-Key`, with `X-ClientBrain-Owner` and `X-ClientBrain-Tool`. It is not put in the URL.
4. Keep those header names. The stored agent sends them. Renaming them without updating the agent breaks phone tools.

The session cookie is `cb_plus_session`. It is HttpOnly. The name is internal.

## Accounts and data

Passwords are stored with scrypt. Sessions last 30 days and the token is stored as a SHA-256 hash.

Leads match on the exact name. A unique prefix or word of at least 3 characters also matches. “Ann” does not match “Joanna.” An ambiguous prefix returns a message and no lead. Creating a lead for an exact name that already exists updates that lead.

On Vercel, signup fails until `DATABASE_URL` is set. That is intentional.

Tables: `owners`, `sessions`, `leads`, `followups`, `activities`, `handoffs`, `voice_agents`. The schema is created on startup in `api/_lib/db.js`.

## Map

| Path | Role |
| --- | --- |
| `src/main.jsx` | Sign-in, voice screen, Leads, Follow-ups |
| `src/styles.css` | Layout |
| `public/pcm-processor.js` | Microphone capture |
| `public/picko.jpg` | Landing mascot |
| `public/picko-mark.jpg` | Logo and favicon |
| `api/auth/` | Signup, login, logout, current account |
| `api/crm.js` | Lead and follow-up tools |
| `api/transfer.js` | Callback request |
| `api/voice-config.js` | Prompt, greeting, voice, tools for the browser |
| `api/voice-token.js` | Short-lived AssemblyAI token |
| `api/create-agent.js` | Create or update the phone agent |
| `api/_lib/voice.js` | The words Picko says and the tool contract |
| `test/` | Name matching, voice contract, per-account book |

## Check before you change hosts

- [ ] New `ASSEMBLYAI_API_KEY` and a new `CLIENTBRAIN_TOOL_SECRET`
- [ ] Postgres that belongs to this app, with `DATABASE_URL_UNPOOLED` set
- [ ] Signup works on the public URL
- [ ] One spoken lead appears under Leads
- [ ] `npm test` passes

## A useful next step

The valuable loop is already here: someone speaks a buyer, and the lead lands in a book. The step that makes it a product for a working agent is sending that lead into the CRM they already use. A phone call that is actually dialed is a later project, and it needs a telephony account of your own.
