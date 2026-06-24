# Jasper's Market WhatsApp Bot — Design and Realization

## 1. Purpose

Jasper's Market is Meta's official sample application for the WhatsApp Business Platform (Cloud API). It is a fictional grocery brand used purely as a vehicle to demonstrate, in working code, four message types a WhatsApp Business app can send:

- Interactive reply-button messages
- Utility templates (transactional/informational, with an image header)
- Limited-time-offer templates (with an expiration countdown and a copy-code button)
- Media card carousel templates (multiple image cards in one message)

It also demonstrates the minimum required plumbing for any Cloud API integration: webhook verification, signature validation, and a delivery/read-receipt-driven follow-up pattern. It is a reference/demo project, not a production system, and is unrelated to the IPX800 home-control project that lives alongside it in this repository.

## 2. Design goals

| Goal | Design response |
|---|---|
| Show the full message-type catalogue with minimal code | One Express route, one dispatch switch, one API wrapper class |
| Be safe to expose on the public internet | HMAC-SHA256 signature verification on every inbound webhook call, performed before the body is trusted |
| Demonstrate a "smart" follow-up without a database | Use Redis purely as a short-TTL marker, not as a session store |
| Keep template content/config out of code logic | Centralize template names, locales, and copy in `constants.js`; centralize request-shape construction in `graph-api.js` |
| Make the demo reproducible by a third party | Ship `template.sh` to (re)create the exact templates and image assets the code references, since templates are server-side objects owned by Meta, not local files |

## 3. Architecture

```
WhatsApp user
     │  taps a button / sends "Get started"
     ▼
Meta Cloud API  ──(POST /webhook, HMAC-signed)──►  app.js (Express, :8080)
     │                                                  │
     │                                                  ├─ verifyRequestSignature()  [HMAC check]
     │                                                  ├─ Conversation.handleMessage()  [for value.messages]
     │                                                  └─ Conversation.handleStatus()   [for value.statuses]
     │                                                          │
     │                                                          ├─ GraphApi.messageWith*()  ──► Meta Graph API
     │                                                          └─ Cache.insert()/remove()  ──► Redis (15s TTL)
     ▼
WhatsApp user  ◄────────────────── reply / template message ───────────────────┘
```

Single process, single port. No persistent database — Redis is the only stateful dependency, and it holds nothing longer than 15 seconds.

## 4. Component realization

### 4.1 `app.js` — transport and security boundary

- `GET /webhook`: implements the Meta webhook handshake. Compares `hub.verify_token` (query param) against `config.verifyToken` (from `.env`); on match, echoes back `hub.challenge` as required by Meta's subscription protocol.
- `POST /webhook`: the single ingress point for all WhatsApp events.
  - Body parsing uses `body-parser`'s `json()` with a `verify` callback (`verifyRequestSignature`), so the raw byte buffer is available for HMAC computation *before* JSON parsing completes. This is realized by reading the `x-hub-signature-256` header, computing `HMAC-SHA256(APP_SECRET, rawBody)`, and throwing if the hex digests don't match — a thrown error here causes Express to reject the request before any handler logic runs.
  - The handler walks Meta's nested webhook payload shape (`entry[].changes[].value`) and routes two kinds of events: `value.messages` (an incoming user message) to `Conversation.handleMessage`, and `value.statuses` (a delivery/read receipt for a message *this app* sent) to `Conversation.handleStatus`.
  - Always responds `200 EVENT_RECEIVED` to Meta regardless of internal outcome, per Meta's webhook contract (Meta retries on non-2xx, not on internal errors after acknowledgment).
- `GET /`: trivial liveness/health endpoint.
- `config.checkEnvVariables()` runs at startup and **warns** (not throws) on missing required vars — the app will still start in a broken state rather than fail fast, which is a deliberate (if debatable) demo-friendly choice: it surfaces in logs but doesn't block local experimentation.

### 4.2 `services/config.js` — configuration realization

Loads `.env` via `dotenv`, exposes a single `Object.freeze()`'d config object so downstream modules cannot mutate shared config at runtime. Required keys (`ACCESS_TOKEN`, `APP_SECRET`, `VERIFY_TOKEN`, `REDIS_HOST`, `REDIS_PORT`) are declared once and checked in a loop, avoiding repeated `if (!process.env.X)` boilerplate.

### 4.3 `services/message.js` / `services/status.js` — payload normalization

Both are minimal adapter classes that convert Meta's raw webhook JSON shape into a small, stable internal shape (`{id, type, senderPhoneNumber}` and `{messageId, status, recipientPhoneNumber}` respectively). This isolates the rest of the codebase from Meta's wire format — if Meta's payload shape changes, only these two files need to change.

`Message.type` is specifically derived from `rawMessage.interactive.button_reply.id` when the incoming message is an interactive button tap, and falls back to `'unknown'` otherwise (e.g. the user's free-text "Get started" message). This `type` value is the routing key for the entire conversation flow.

### 4.4 `services/constants.js` — content/config registry

Holds all user-facing copy (`APP_DEFAULT_MESSAGE`, `APP_TRY_ANOTHER_MESSAGE`) and the three reply-button identifiers (`REPLY_INTERACTIVE_MEDIA_ID`, `REPLY_MEDIA_CAROUSEL_ID`, `REPLY_OFFER_ID`) plus their button labels. Realizing these as named constants rather than inline strings means `conversation.js`'s `switch` statement and `app.js`'s default message stay in sync by construction.

### 4.5 `services/conversation.js` — the conversation state machine

This is the realization of the actual demo logic, implemented without any persisted per-user state:

- **`handleMessage(senderPhoneNumberId, rawMessage)`**: wraps the raw payload in a `Message`, then `switch`es on `message.type`:
  - Unknown/first message → `sendTryOutDemoMessage`, which sends an interactive message with three reply buttons (Shop online / Get recipe ideas / Current promo).
  - `REPLY_INTERACTIVE_MEDIA_ID` → sends the `grocery_delivery_utility` template.
  - `REPLY_MEDIA_CAROUSEL_ID` → sends the `recipe_media_carousel` template.
  - `REPLY_OFFER_ID` → sends the `strawberries_limited_offer` template (with a hardcoded `BERRIES20` offer code).
  - In every templated branch, the **outgoing** message's id (returned by the Graph API call) is recorded via `markMessageForFollowUp`, which is just `Cache.insert(messageId)` with an implicit 15s TTL.
- **`handleStatus(senderPhoneNumberId, rawStatus)`**: wraps the raw payload in a `Status`, and only proceeds for `delivered` or `read` statuses (ignoring `sent`/`failed`). It then calls `Cache.remove(status.messageId)` — if that message ID was previously marked (i.e. this status update corresponds to one of the templated messages the bot just sent), it sends a generic "Anything else we can help with?" follow-up.

This is the core design trick of the whole demo: instead of maintaining a conversation/session object per user, the bot uses **Redis as a transient correlation table** between "a templated message was sent" and "that same message was just delivered/read," with a 15-second window long enough to cover normal Cloud API latency. If the read receipt never arrives within 15 seconds, the key simply expires and no follow-up is sent — a deliberate trade-off favoring simplicity over guaranteed delivery of the follow-up.

### 4.6 `services/graph-api.js` — outbound message construction

A static class (no instance state) wrapping `facebook-nodejs-business-sdk`'s `FacebookAdsApi`. Every public method funnels through a private `#makeApiCall`, which:

1. If a `messageId` is supplied, first fires a "mark as read + show typing indicator" call — this is realized as a *separate* API call before the actual content, matching the UX pattern WhatsApp's own client uses (read receipt, then typing dots, then message).
2. Sends the actual content as a second API call to `POST /{senderPhoneNumberId}/messages`.

Four message-shape builders are realized here, each assembling the Cloud API JSON body Meta's API expects:

- `messageWithInteractiveReply` — `type: "interactive"`, button array built from a generic `{id, title}` list (decouples this builder from the specific three demo buttons — it would work for any reply-button set).
- `messageWithUtilityTemplate` — `type: "template"`, single image header component.
- `messageWithLimitedTimeOfferTemplate` — adds a `limited_time_offer` component with `expiration_time_ms` computed at send-time as `now + 48h`, plus a `copy_code` button component carrying the offer code.
- `messageWithMediaCardCarousel` — builds a `carousel` component with one card per image link, each card given an explicit `card_index`.

Centralizing all four shapes here means `conversation.js` never constructs raw Graph API JSON — it only supplies business parameters (template name, locale, image links, offer code).

### 4.7 `services/redis.js` — minimal cache realization

A two-method static class: `insert(key)` sets an empty value with a 15-second expiry (chosen because the redis client used does not support setting TTL atomically on set members, so a plain key/empty-value hash entry is used as a substitute for a TTL'd set); `remove(key)` does a `DEL` and returns whether anything was actually deleted (`resp > 0`), which is how `handleStatus` knows whether *this* status event was the one being waited on.

### 4.8 `template.sh` — provisioning script (realization of the "Meta owns templates" constraint)

WhatsApp marketing templates are server-side objects that must be created and (for marketing category) approved on the WABA before the app can reference them by name. `template.sh` realizes the one-time provisioning step the running Node app depends on but never performs itself:

1. Downloads four fixed demo images from Meta's CDN to `public/`.
2. For each image, calls the Graph API `/uploads` resumable-upload endpoint to obtain a session, then `PUT`s the binary to that session to obtain a reusable image **handle**.
3. Submits three `POST {WABA_ID}/message_templates` calls — one per template referenced in `graph-api.js`/`conversation.js` — embedding the handles as header image examples.

This script is a deployment/setup-time dependency, not a runtime one: once the templates exist on the WABA (status `APPROVED`), the Node app only ever references them by name and locale.

## 5. Security realization

- **Signature verification** is mandatory and enforced inline in the body-parser pipeline rather than as an optional middleware — there's no code path where an unsigned/mis-signed POST reaches `Conversation.handleMessage`.
- Secrets (`ACCESS_TOKEN`, `APP_SECRET`, `VERIFY_TOKEN`) live only in `.env`, never in source; `.sample.env` documents the required shape without real values.
- No user input is parsed as anything other than an opaque routing key (`button_reply.id`) — there is no free-text command parsing, so there's no injection surface in the conversation logic itself.

## 6. Known limitations (as-built, intentionally out of scope for a demo)

- No persistence beyond Redis's 15s TTL — a server restart or Redis flush silently drops any in-flight follow-up correlation; this only ever causes a missed "anything else?" message, never an error.
- No retry/backoff around Graph API calls in `graph-api.js` — a failed `api.call` simply logs and rethrows, which surfaces as an unhandled rejection at the call site in `conversation.js` (no try/catch there).
- `config.checkEnvVariables()` only warns; the app does not refuse to start with missing credentials, so a misconfigured deployment fails at first webhook call rather than at boot.
- `template.sh` uses `stat -f%z` (BSD/macOS) for file size — needs `stat -c%s` substitution to run on Linux.
- Single hardcoded offer code (`BERRIES20`) and fixed image URLs — by design, since this is a fixed demo scenario, not a templated/configurable promotions engine.
