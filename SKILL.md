---
name: gowa-whatsapp-api
description: |
  GOWA (Go WhatsApp) REST API for WhatsApp automation via gowa.megawebs.com.
  Triggers: "whatsapp", "gowa", "send whatsapp", "whatsapp message", "whatsapp chats",
  "check whatsapp", "whatsapp group", "group participants", "send message whatsapp",
  "list chats", "read messages", "whatsapp contacts", "whatsapp outreach", "bulk whatsapp",
  "whatsapp check number", "is on whatsapp", "gowa api", "gowa curl".

  Use when: (1) Listing WhatsApp chats and groups, (2) Reading chat history,
  (3) Sending text or media to people or groups, (4) Inspecting group state,
  (5) Checking whether a phone exists on WhatsApp, (6) Running GOWA REST API calls
  directly, (7) Handling self-hosted GOWA multi-device workflows, (8) Bulk outreach.

  ALWAYS use this skill instead of the WhatsApp Go MCP tool.
  ALWAYS prefer direct curl calls to the GOWA REST API over any MCP wrapper.
  For the managed shared instance, default to gowa.megawebs.com through the CORS proxy.
---

# GOWA WhatsApp API

Operational reference for the GOWA REST API, based on repeated working usage against the managed instance plus prior self-hosted GOWA notes.

This skill has two modes:

1. Managed shared instance at `https://gowa.megawebs.com`
2. Self-hosted GOWA deployments exposing `/app/*`, `/send/*`, `/group/*`, `/chat/*`, etc.

The main production path for this environment is the managed shared instance.

## Fast Rules

1. Always use direct `curl`.
2. For the shared managed instance, route requests through `https://cors.trigox.workers.dev`.
3. Never use the WhatsApp Go MCP tools.
4. On the managed instance, use `device_id=sami`.
5. On managed GET requests, pass `device_id=sami` in the query string.
6. On managed POST requests, pass `"device_id": "sami"` in the JSON body.
7. Use `34XXXXXXXXX` for bare phone numbers.
8. Use `34XXXXXXXXX@s.whatsapp.net` for person JIDs.
9. Use `120363XXXXXXXXXXXX@g.us` for group JIDs.
10. When reading chat history, use the full JID in the path.

## Base Configuration

```text
MANAGED_BASE_URL=https://gowa.megawebs.com
MANAGED_PROXY=https://cors.trigox.workers.dev
MANAGED_DEVICE_ID=sami
```

Managed request template:

```bash
curl -s "https://cors.trigox.workers.dev/https://gowa.megawebs.com/..."
```

## Number And JID Formats

People:

- Bare phone: `34642609188`
- Person JID: `34642609188@s.whatsapp.net`

Groups:

- Group JID: `120363411006743584@g.us`

Important:

- Sending endpoints often accept either bare phone or full JID.
- Read-history endpoints require the full JID in the URL path.
- Group endpoints generally require the group JID.

## What Is Actually Verified vs Cataloged

### Operationally Verified On The Managed Shared Instance

- `GET /chats`
- `GET /chat/{JID}/messages`
- `POST /send/message`
- `POST /send/text`
- `GET /user/check`
- `GET /group/info`
- `GET /group/participants`

### Verified Behavioral Findings From Prior Runs

- The path for message history is `/chat/{JID}/messages`, not any `messages?chat_id=` variant.
- Sending to groups works by passing the group JID in the `phone` field.
- Image delivery depends heavily on whether GOWA's server-side fetcher can reach the remote image URL.
- Placeholder hosts like `picsum.photos` have worked in practice when stricter hosts failed.
- Some real-photo hosts and CDNs have failed with 403 or format errors when fetched by GOWA.
- Self-hosted GOWA supports multi-device sessions via a `device` query parameter on app endpoints.
- Calling self-hosted `/app/login` without a different `device` can return `ALREADY_LOGGED_IN` for the default session.

### Cataloged From Prior GOWA UI / Self-Hosted Notes

These were observed in the GOWA app UI and prior notes, but not all were re-executed in this turn. Treat them as the broader endpoint map for self-hosted GOWA:

- `/app/login`
- `/app/login-with-code`
- `/app/logout`
- `/app/reconnect`
- `/app/status`
- `/app/devices`
- `/send/image`
- `/send/file`
- `/send/video`
- `/send/sticker`
- `/send/contact`
- `/send/location`
- `/send/audio`
- `/send/poll`
- `/send/presence`
- `/send/chat-presence`
- `/send/link`
- `/message/delete`
- `/message/revoke`
- `/message/react`
- `/message/update`
- `/message/mark-read`
- `/group/list`
- `/group/create`
- `/group/join`
- `/group/preview`
- `/group/participants/manage`
- `/group/photo`
- `/group/name`
- `/group/locked`
- `/group/announce`
- `/group/topic`
- `/group/invite-link`
- `/newsletter/list`
- `/account/avatar`
- `/account/avatar/change`
- `/account/pushname`
- `/account/user-info`
- `/account/business-profile`
- `/account/privacy`
- `/contacts/my`
- `/chat/pin`
- `/chat/disappearing-messages`

## Managed Shared Instance: Working Endpoints

### 1. List Chats

Returns chats, both direct and group, newest first.

```bash
curl -s \
  "https://cors.trigox.workers.dev/https://gowa.megawebs.com/chats?device_id=sami&limit=100&offset=0"
```

Parameters:

- `device_id` required
- `limit` optional, practical max observed: `100`
- `offset` optional, zero-based

Response shape:

```json
{
  "results": {
    "data": [
      {
        "jid": "34642609188@s.whatsapp.net",
        "name": "KITTY ZHU",
        "last_message_time": "2026-02-27T10:43:00Z"
      }
    ],
    "pagination": {
      "total": 801,
      "limit": 100,
      "offset": 0
    }
  }
}
```

Pagination loop:

```bash
for offset in 0 100 200 300 400 500 600 700 800; do
  curl -s \
    "https://cors.trigox.workers.dev/https://gowa.megawebs.com/chats?device_id=sami&limit=100&offset=$offset"
done
```

List only groups:

```bash
curl -s \
  "https://cors.trigox.workers.dev/https://gowa.megawebs.com/chats?device_id=sami&limit=100&offset=0" \
  | jq -r '.results.data[] | select(.jid | endswith("@g.us")) | [.name, .jid] | @tsv'
```

### 2. Read Messages From A Chat

Correct path:

```bash
curl -s \
  "https://cors.trigox.workers.dev/https://gowa.megawebs.com/chat/34642609188@s.whatsapp.net/messages?device_id=sami&limit=100"
```

Group example:

```bash
curl -s \
  "https://cors.trigox.workers.dev/https://gowa.megawebs.com/chat/120363411006743584@g.us/messages?device_id=sami&limit=100"
```

Parameters:

- `device_id` required
- `limit` optional
- `offset` optional

Critical path rules:

- Correct: `/chat/{JID}/messages`
- Wrong: `/chat/messages/{JID}`
- Wrong: `/chat/messages?jid={JID}`
- Wrong: `/messages?chat_id={JID}`
- Wrong: `/chat/{JID}/history`

Response shape:

```json
{
  "results": {
    "data": [
      {
        "timestamp": "2026-02-27T12:35:00Z",
        "content": "Message text here",
        "from": "34679794037@s.whatsapp.net",
        "is_from_me": true
      }
    ],
    "pagination": {
      "total": 353
    }
  }
}
```

### 3. Send Text Message

Preferred working endpoint:

```bash
curl -s -X POST \
  "https://cors.trigox.workers.dev/https://gowa.megawebs.com/send/message" \
  -H "Content-Type: application/json" \
  -d '{
    "device_id": "sami",
    "phone": "34642609188@s.whatsapp.net",
    "message": "Hello from GOWA API",
    "is_forwarded": false
  }'
```

Alternative endpoint:

```bash
curl -s -X POST \
  "https://cors.trigox.workers.dev/https://gowa.megawebs.com/send/text" \
  -H "Content-Type: application/json" \
  -d '{
    "device_id": "sami",
    "phone": "34642609188",
    "message": "Hello from GOWA API"
  }'
```

Parameters:

- `device_id` required
- `phone` required
- `message` required
- `is_forwarded` optional

Success shape:

```json
{
  "results": {
    "message_id": "3EB0010200C5DABF137B09",
    "status": "sent"
  }
}
```

Send to group:

```bash
curl -s -X POST \
  "https://cors.trigox.workers.dev/https://gowa.megawebs.com/send/message" \
  -H "Content-Type: application/json" \
  -d '{
    "device_id": "sami",
    "phone": "120363411006743584@g.us",
    "message": "Group message here"
  }'
```

### 4. Check Whether A Number Is On WhatsApp

```bash
curl -s \
  "https://cors.trigox.workers.dev/https://gowa.megawebs.com/user/check?device_id=sami&phone=34642609188"
```

Parameters:

- `device_id` required
- `phone` required, bare digits only

Typical use:

- check availability before outreach
- convert a plain phone list into valid WhatsApp targets

### 5. Group Info

```bash
curl -s \
  "https://cors.trigox.workers.dev/https://gowa.megawebs.com/group/info?device_id=sami&group_id=120363411006743584@g.us"
```

### 6. Group Participants

```bash
curl -s \
  "https://cors.trigox.workers.dev/https://gowa.megawebs.com/group/participants?device_id=sami&group_id=120363411006743584@g.us"
```

Response shape:

```json
{
  "results": {
    "data": [
      {
        "jid": "34679794037@s.whatsapp.net",
        "is_admin": true,
        "is_super_admin": true
      }
    ]
  }
}
```

## Self-Hosted GOWA: Multi-Device Notes

These notes matter when the user is working against their own deployed GOWA rather than the shared `gowa.megawebs.com` instance.

### Device Model

Self-hosted app endpoints use a `device` query parameter, not the managed-instance `device_id=sami` convention.

Examples:

```bash
curl -s "http://your-host/app/login?device=account2"
curl -s "http://your-host/app/status?device=account2"
curl -s "http://your-host/app/devices"
```

Key finding from prior runs:

- Hitting `/app/login` with the default device when that device is already connected can return:

```json
{
  "code": "ALREADY_LOGGED_IN",
  "message": "you are already logged in."
}
```

Meaning:

- do not log out the default device unless explicitly requested
- add a second session by using a fresh device name like `account2`

### Self-Hosted App Endpoints

Observed endpoints:

- `GET /app/login?device=...`
- `POST /app/login-with-code`
- `POST /app/logout`
- `POST /app/reconnect`
- `GET /app/status?device=...`
- `GET /app/devices`

Typical sequence:

1. `GET /app/login?device=account2`
2. scan QR or use pairing code
3. `GET /app/status?device=account2`
4. `GET /app/devices`

## Webhooks

Self-hosted GOWA supports push delivery for new messages and other events. It is not limited to polling.

This is the correct model when you control the server:

- use webhooks for inbound events
- use REST queries for backfill, replay, manual inspection, and admin actions

### Core Config

CLI flags:

```bash
./whatsapp rest \
  --webhook="https://your-app.example/webhooks/gowa" \
  --webhook-secret="super-secret-key" \
  --webhook-events="message,message.ack"
```

Environment variables:

```bash
WHATSAPP_WEBHOOK=https://your-app.example/webhooks/gowa
WHATSAPP_WEBHOOK_SECRET=super-secret-key
WHATSAPP_WEBHOOK_EVENTS=message,message.ack
WHATSAPP_WEBHOOK_INSECURE_SKIP_VERIFY=false
```

Important findings:

- if `WHATSAPP_WEBHOOK_EVENTS` is empty, all supported events are forwarded
- webhook delivery is a server-level self-hosted feature, not something to assume on a shared multi-tenant instance
- webhook requests include an HMAC header using the configured secret

### Relevant Events

Known documented event names:

- `message`
- `message.reaction`
- `message.revoked`
- `message.edited`
- `message.ack`
- `message.deleted`
- `group.participants`
- `group.joined`
- `newsletter.joined`
- `newsletter.left`
- `newsletter.message`
- `newsletter.mute`
- `call.offer`

For "notify me on new messages", the minimum useful filter is:

```bash
WHATSAPP_WEBHOOK_EVENTS=message
```

For a practical bot / CRM integration:

```bash
WHATSAPP_WEBHOOK_EVENTS=message,message.ack,message.reaction,group.participants
```

### TLS Note

If webhook delivery fails due to TLS verification on tunnels or self-signed certs:

```bash
WHATSAPP_WEBHOOK_INSECURE_SKIP_VERIFY=true
```

Only use that for development, tunnels, or controlled internal networks.

### Recommended Architecture

Use both:

1. Webhook receiver for immediate inbound events.
2. REST API for:
   - chat history sync
   - missed-event recovery
   - manual group inspection
   - sending outbound messages

Do not build a system that relies only on polling if you control the GOWA deployment.

## Media Sending Notes

### Image Sending Reality

The GOWA UI advertises endpoints for:

- `POST /send/image`
- `POST /send/file`
- `POST /send/video`
- `POST /send/sticker`
- `POST /send/audio`
- `POST /send/contact`
- `POST /send/location`
- `POST /send/poll`
- `POST /send/link`

But the practical constraint is not just the endpoint. The remote media URL must be fetchable by GOWA's server-side fetcher.

### Prior Image Delivery Findings

From prior runs:

- placeholder image hosts such as `picsum.photos` worked
- some stricter hosts blocked the server-side fetch
- several real-photo hosts or CDN links failed with `403` or format errors
- Wikipedia image URLs and some TheFork / restaurant CDN URLs were unreliable in this flow

Best practical advice:

- use raw GitHub file URLs, Cloudflare R2, ImgBB, or another simple public host
- avoid hosts that require browser cookies, referer, or anti-bot headers
- use direct image URLs ending in a recognizable file

Recommended pattern:

1. test the image URL with normal `curl -I`
2. if possible, test whether the file is directly downloadable without redirects or auth
3. if image sending fails, move the asset to a simpler public host

### Media Workflow Template

If the user explicitly needs media sending, first verify which form the target GOWA instance expects.

General self-hosted pattern:

```bash
curl -s -X POST "http://your-host/send/image?device=account2" \
  -H "Content-Type: application/json" \
  -d '{
    "phone": "34642609188@s.whatsapp.net",
    "image_url": "https://your-public-host/image.jpg",
    "caption": "Image caption"
  }'
```

Managed shared instance:

- verify current payload shape before assuming field names
- if unverified, say so and inspect the live instance or UI docs first

## Full Endpoint Catalog

This section is the broadest known GOWA map, combining working managed endpoints with endpoints exposed by the self-hosted GOWA app UI.

### App

- `GET /app/login`
- `POST /app/login-with-code`
- `POST /app/logout`
- `POST /app/reconnect`
- `GET /app/status`
- `GET /app/devices`

### Send

- `POST /send/message`
- `POST /send/text`
- `POST /send/image`
- `POST /send/file`
- `POST /send/video`
- `POST /send/sticker`
- `POST /send/contact`
- `POST /send/location`
- `POST /send/audio`
- `POST /send/poll`
- `POST /send/presence`
- `POST /send/chat-presence`
- `POST /send/link`

### Message

- `POST /message/delete`
- `POST /message/revoke`
- `POST /message/react`
- `POST /message/update`
- `POST /message/mark-read`

### Group

- `GET /group/list`
- `POST /group/create`
- `POST /group/join`
- `GET /group/preview`
- `GET /group/info`
- `GET /group/participants`
- `POST /group/participants/manage`
- `POST /group/photo`
- `POST /group/name`
- `POST /group/locked`
- `POST /group/announce`
- `POST /group/topic`
- `GET /group/invite-link`

### Newsletter

- `GET /newsletter/list`

### Account

- `GET /account/avatar`
- `POST /account/avatar/change`
- `POST /account/pushname`
- `GET /account/user-info`
- `GET /account/business-profile`
- `GET /account/privacy`

### Contacts

- `GET /contacts/my`

### Chat

- `POST /chat/pin`
- `POST /chat/disappearing-messages`
- `GET /chats`
- `GET /chat/{JID}/messages`

### User

- `GET /user/check`

## Endpoints That Have Been Wrong In Practice

Do not waste time on these variants:

- `GET /messages?device_id=...&chat_id=...`
- `GET /message/list?device_id=...`
- `GET /chat/history?device_id=...&jid=...`
- `GET /chat/{JID}/history`
- `GET /chat/messages/{JID}`
- `GET /chat/messages?jid={JID}`
- `GET /contacts?device_id=...`
- `GET /contact/list?device_id=...`
- `GET /user/contacts?device_id=...`
- `GET /groups/{JID}?device_id=...`
- `GET /group/list?device_id=...` on the managed shared instance unless you have confirmed it there

## Common Workflows

### Workflow A: Find A Chat Then Read It

```bash
curl -s \
  "https://cors.trigox.workers.dev/https://gowa.megawebs.com/chats?device_id=sami&limit=100&offset=0" \
  | jq -r '.results.data[] | [.name, .jid] | @tsv'

curl -s \
  "https://cors.trigox.workers.dev/https://gowa.megawebs.com/chat/34642609188@s.whatsapp.net/messages?device_id=sami&limit=50"
```

### Workflow B: Outreach With Verification First

```bash
PHONE=34642609188
MESSAGE="Hello from GOWA"

curl -s \
  "https://cors.trigox.workers.dev/https://gowa.megawebs.com/user/check?device_id=sami&phone=$PHONE"

curl -s \
  "https://cors.trigox.workers.dev/https://gowa.megawebs.com/chat/${PHONE}@s.whatsapp.net/messages?device_id=sami&limit=5"

curl -s -X POST \
  "https://cors.trigox.workers.dev/https://gowa.megawebs.com/send/message" \
  -H "Content-Type: application/json" \
  -d "{\"device_id\":\"sami\",\"phone\":\"${PHONE}@s.whatsapp.net\",\"message\":\"$MESSAGE\"}"
```

### Workflow C: Inspect A Group

```bash
GROUP_JID="120363411006743584@g.us"

curl -s \
  "https://cors.trigox.workers.dev/https://gowa.megawebs.com/group/info?device_id=sami&group_id=$GROUP_JID"

curl -s \
  "https://cors.trigox.workers.dev/https://gowa.megawebs.com/group/participants?device_id=sami&group_id=$GROUP_JID"

curl -s \
  "https://cors.trigox.workers.dev/https://gowa.megawebs.com/chat/$GROUP_JID/messages?device_id=sami&limit=100"
```

### Workflow D: Bulk Send Safely

```bash
PHONES="34611111111 34622222222 34633333333"
MESSAGE="Your bulk message here"

for phone in $PHONES; do
  check=$(curl -s \
    "https://cors.trigox.workers.dev/https://gowa.megawebs.com/user/check?device_id=sami&phone=$phone")

  if echo "$check" | jq -e '.results.jid? // .results.data?.jid? // .results.data?[0]?.jid?' >/dev/null 2>&1; then
    curl -s -X POST \
      "https://cors.trigox.workers.dev/https://gowa.megawebs.com/send/message" \
      -H "Content-Type: application/json" \
      -d "{\"device_id\":\"sami\",\"phone\":\"${phone}@s.whatsapp.net\",\"message\":\"$MESSAGE\"}"
    echo "Sent to $phone"
    sleep 2
  else
    echo "Skipped $phone"
  fi
done
```

### Workflow E: Self-Hosted Second Device

```bash
HOST="http://your-host"

curl -s "$HOST/app/login?device=account2"
curl -s "$HOST/app/status?device=account2"
curl -s "$HOST/app/devices"
```

## Error Handling

| Error | Meaning | Fix |
|---|---|---|
| `Cannot GET /...` | Wrong endpoint | Use the verified route map |
| empty response through proxy | Proxy issue or upstream issue | retry direct host if allowed |
| `device not found` | wrong device ID | use `device_id=sami` on managed instance |
| `ALREADY_LOGGED_IN` | default self-hosted device already connected | use a fresh `device=...` value |
| `not on whatsapp` | target number not registered | skip or verify number |
| `403` while sending image | media host blocks server-side fetcher | rehost image on simpler public URL |
| format/media error | unsupported media fetch or payload mismatch | verify direct URL and expected payload |

## Quick Reference

| Action | Method | Endpoint | Notes |
|---|---|---|---|
| List chats | GET | `/chats` | managed: `device_id=sami` |
| Read chat history | GET | `/chat/{JID}/messages` | full JID required |
| Send text | POST | `/send/message` | managed working path |
| Send text alt | POST | `/send/text` | simpler variant |
| Check number | GET | `/user/check` | bare phone digits |
| Group info | GET | `/group/info` | needs group JID |
| Group participants | GET | `/group/participants` | needs group JID |
| Login self-hosted device | GET | `/app/login?device=...` | self-hosted only |
| Device status | GET | `/app/status?device=...` | self-hosted only |
| All devices | GET | `/app/devices` | self-hosted only |
| Send image | POST | `/send/image` | payload must be verified per instance |
| Send file | POST | `/send/file` | self-hosted catalog |
| Join group | POST | `/group/join` | self-hosted catalog |

## Default Agent Behavior

When asked to do anything with GOWA:

1. Prefer the managed shared instance unless the user clearly points to a self-hosted base URL.
2. If the task is chats, message history, user checks, group info, or plain text sending, use the managed verified endpoints above.
3. If the task is self-hosted login, multi-device, QR, or app status, switch to the `/app/*` model with `device=...`.
4. If the task is media sending, do not trust the remote image host blindly.
5. If a route fails, do not invent a new variant. Compare against the verified map first.

## Hard Rule

Do not use the WhatsApp Go MCP server for this workflow. Use direct `curl` to GOWA.
