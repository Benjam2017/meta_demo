# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Meta's official "Jasper's Market" sample app (`fbsamples/whatsapp-business-jaspers-market`) — a single Express server demonstrating WhatsApp Business Platform features: interactive reply buttons, utility templates, limited-time-offer templates, and media card carousels. It is a demo, not a production app; there is no relation to the IPX800 home-control project in the parent directories.

## Commands

```bash
npm install              # install dependencies
cp .sample.env .env       # then fill in ACCESS_TOKEN, APP_SECRET, APP_ID, VERIFY_TOKEN; REDIS_HOST/REDIS_PORT optional (default localhost:6379)
redis-server --daemonize yes   # Redis must be running before starting the app
node app.js               # or: npm start
```

There is no test suite (`npm test` just echoes a placeholder) and no lint/build step.

For local development, expose the server with a tunnel (e.g. `ngrok http 8080`) and register the resulting HTTPS URL + `VERIFY_TOKEN` as the webhook in the Meta App's WhatsApp > Configuration tab, subscribed to the `messages` field.

`template.sh` is a one-time setup script that uploads the four image assets used by the templates to Meta's Asset Manager and prints the resulting template definitions — needs `APPTOKEN`, `APPID`, `WABAID`, `APIVERSION` filled in at the top before running.

## Architecture

Single entry point `app.js`:
- `GET /webhook` — verification handshake (checks `hub.verify_token` against `config.verifyToken`, echoes `hub.challenge`).
- `POST /webhook` — receives Meta Cloud API webhook events. Verified via HMAC SHA-256 over the raw body (`x-hub-signature-256` header vs. `APP_SECRET`) in the body-parser `verify` callback — an invalid signature throws inside that callback. Dispatches each `entry[].changes[].value` to `Conversation.handleMessage` (for `value.messages`) or `Conversation.handleStatus` (for `value.statuses`).

`services/` holds the rest of the logic:
- `config.js` — loads `.env` via dotenv, exposes a frozen config object, warns (does not throw) on missing required env vars.
- `constants.js` — all user-facing copy and the button-reply IDs used as a routing key.
- `message.js` / `status.js` — thin wrappers that normalize a raw webhook payload into `{id, type, senderPhoneNumber}` / `{messageId, status, recipientPhoneNumber}`.
- `conversation.js` — the actual conversation flow, implemented as a `switch` on the incoming button-reply ID (`Message.type`):
  - default (any first/unrecognized message) → sends the three-button "Try out the demo" interactive message.
  - each button ID → sends the corresponding template (utility image, limited-time-offer, or media carousel) via `GraphApi`, then calls `markMessageForFollowUp` to cache the *outgoing* message's ID in Redis with a 15s TTL.
  - `handleStatus` only reacts to `delivered`/`read` statuses; if the status's message ID is found (and removed) from the Redis cache, it sends a follow-up "anything else?" message. This is how the app knows to follow up only after the user has actually seen the demo message, without persisting any other conversation state.
- `graph-api.js` — wraps `facebook-nodejs-business-sdk`'s `FacebookAdsApi` to call the WhatsApp `POST /{phone_number_id}/messages` endpoint. Every send first fires a "mark as read + typing indicator" call (when a `messageId` is available), then the actual message. Four message shapes are built here: interactive reply buttons, utility template, limited-time-offer template (sets a 48-hour expiration), and media card carousel template.
- `redis.js` — minimal Cache wrapper (`insert`/`remove`) used purely as a short-TTL "was this message delivered/read yet" flag; not a general session store.

Conversation state is **not** persisted beyond the 15-second Redis TTL — each incoming message is handled statelessly based on its button-reply ID alone; there's no per-user session object.
