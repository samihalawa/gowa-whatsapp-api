---
name: gowa-whatsapp-api
description: |
  GOWA (Go WhatsApp) REST API for WhatsApp automation via gowa.megawebs.com.
  Triggers: "whatsapp", "gowa", "send whatsapp", "whatsapp message", "whatsapp chats",
  "check whatsapp", "whatsapp group", "group participants", "send message whatsapp",
  "list chats", "read messages", "whatsapp contacts", "whatsapp outreach", "bulk whatsapp",
  "whatsapp check number", "is on whatsapp", "gowa api", "gowa curl".

  Use when: (1) Listing WhatsApp chats/conversations, (2) Reading messages from any chat,
  (3) Sending WhatsApp messages to individuals or groups, (4) Getting group info/participants,
  (5) Checking if a phone number is on WhatsApp, (6) Any WhatsApp automation via REST API,
  (7) Cross-referencing contacts with WhatsApp, (8) Bulk messaging campaigns.

  ALWAYS use this skill instead of the WhatsApp Go MCP tool (which is unreliable).
  ALWAYS use this skill when the user mentions WhatsApp, GOWA, or messaging contacts.
  ALWAYS prefer direct curl calls to the GOWA REST API over any MCP wrapper.
---

# GOWA WhatsApp REST API

Complete reference for the GOWA (Go WhatsApp) REST API at `gowa.megawebs.com`.

## Base Configuration

```text
BASE_URL: https://gowa.megawebs.com
CORS_PROXY: https://cors.trigox.workers.dev
DEVICE_ID: sami
```

## Required Workflow

1. Use direct `curl` calls through the CORS proxy for all GOWA requests.
2. For GET requests, include `device_id=sami` in the query string.
3. For POST requests, send `device_id: "sami"` in the JSON body.
4. Use `34XXXXXXXXX` format for phone numbers and full JIDs when reading chat history.
5. Do not use the WhatsApp Go MCP tools.

## Critical Rules

1. ALWAYS use `cors.trigox.workers.dev` proxy for all calls: `curl -s "https://cors.trigox.workers.dev/https://gowa.megawebs.com/..."`
2. NEVER use the WhatsApp Go MCP tools. Always curl directly.
3. `device_id=sami` is required on ALL GET requests as a query parameter.
4. For POST requests, `device_id` goes in the JSON body OR as `X-Device-Id` header.
5. Phone numbers use format `34XXXXXXXXX` (country code + number, no `+` prefix).
6. JID format for individuals: `34XXXXXXXXX@s.whatsapp.net`
7. JID format for groups: `120363XXXXXXXXXXXX@g.us`

## Verified Endpoints

### 1. List Chats (GET)

Returns all conversations (individual + group), sorted by most recent.

```bash
curl -s "https://cors.trigox.workers.dev/https://gowa.megawebs.com/chats?device_id=sami&limit=100&offset=0"
```

Parameters:
- `device_id` (required): Device identifier
- `limit` (optional): Max results per page (max 100, default 25)
- `offset` (optional): Pagination offset (0-based)

Response structure:

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

Pagination example:

```bash
for offset in 0 100 200 300 400 500 600 700 800; do
  curl -s "https://cors.trigox.workers.dev/https://gowa.megawebs.com/chats?device_id=sami&limit=100&offset=$offset"
done
```

### 2. Read Messages From Chat (GET)

The JID goes in the URL path before `/messages`.

```bash
# Individual chat
curl -s "https://cors.trigox.workers.dev/https://gowa.megawebs.com/chat/34611158350@s.whatsapp.net/messages?device_id=sami&limit=100"

# Group chat
curl -s "https://cors.trigox.workers.dev/https://gowa.megawebs.com/chat/120363423943239814@g.us/messages?device_id=sami&limit=100"
```

Critical path format:
- Correct: `/chat/{JID}/messages`
- Wrong: `/chat/messages/{JID}`
- Wrong: `/chat/messages?jid={JID}`
- Wrong: `/messages?chat_id={JID}`

Parameters:
- `device_id` (required): Device identifier
- `limit` (optional): Max messages to return
- `offset` (optional): Pagination offset

Response structure:

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

### 3. Send Message (POST)

Working endpoint:

```bash
curl -s -X POST "https://cors.trigox.workers.dev/https://gowa.megawebs.com/send/message" \
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
curl -s -X POST "https://cors.trigox.workers.dev/https://gowa.megawebs.com/send/text" \
  -H "Content-Type: application/json" \
  -d '{
    "device_id": "sami",
    "phone": "34642609188",
    "message": "Hello from GOWA API"
  }'
```

Parameters:
- `device_id` (required): `"sami"`
- `phone` (required): Accepts `34XXXXXXXXX` or `34XXXXXXXXX@s.whatsapp.net`
- `message` (required): Text body, supports `\n`
- `is_forwarded` (optional): Boolean

Response on success:

```json
{
  "results": {
    "message_id": "3EB0010200C5DABF137B09",
    "status": "sent"
  }
}
```

Sending to groups:

```bash
curl -s -X POST "https://cors.trigox.workers.dev/https://gowa.megawebs.com/send/message" \
  -H "Content-Type: application/json" \
  -d '{
    "device_id": "sami",
    "phone": "120363423943239814@g.us",
    "message": "Group message here"
  }'
```

### 4. Check If Phone Is On WhatsApp (GET)

```bash
curl -s "https://cors.trigox.workers.dev/https://gowa.megawebs.com/user/check?device_id=sami&phone=34642609188"
```

Parameters:
- `device_id` (required): Device identifier
- `phone` (required): `34XXXXXXXXX` without suffix

### 5. Group Info (GET)

```bash
curl -s "https://cors.trigox.workers.dev/https://gowa.megawebs.com/group/info?device_id=sami&group_id=120363423943239814@g.us"
```

### 6. Group Participants (GET)

```bash
curl -s "https://cors.trigox.workers.dev/https://gowa.megawebs.com/group/participants?device_id=sami&group_id=120363423943239814@g.us"
```

Response structure:

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

## Endpoints That Do Not Exist

Never attempt these:

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
- `GET /group/list?device_id=...`

## Common Workflows

### Workflow A: Check And Read Chats

```bash
curl -s "https://cors.trigox.workers.dev/https://gowa.megawebs.com/chats?device_id=sami&limit=20"
curl -s "https://cors.trigox.workers.dev/https://gowa.megawebs.com/chat/34642609188@s.whatsapp.net/messages?device_id=sami&limit=50"
```

### Workflow B: Contact Outreach

```bash
curl -s "https://cors.trigox.workers.dev/https://gowa.megawebs.com/user/check?device_id=sami&phone=34XXXXXXXXX"
curl -s "https://cors.trigox.workers.dev/https://gowa.megawebs.com/chat/34XXXXXXXXX@s.whatsapp.net/messages?device_id=sami&limit=5"
curl -s -X POST "https://cors.trigox.workers.dev/https://gowa.megawebs.com/send/message" \
  -H "Content-Type: application/json" \
  -d '{"device_id":"sami","phone":"34XXXXXXXXX@s.whatsapp.net","message":"Your message here"}'
```

### Workflow C: Group Analysis

```bash
curl -s "https://cors.trigox.workers.dev/https://gowa.megawebs.com/chats?device_id=sami&limit=100" | python3 -c "
import json,sys
data=json.load(sys.stdin)
for c in data['results']['data']:
    if '@g.us' in c['jid']:
        print(f\"{c['name']:40s} | {c['jid']}\")"

curl -s "https://cors.trigox.workers.dev/https://gowa.megawebs.com/group/info?device_id=sami&group_id=GROUP_JID"
curl -s "https://cors.trigox.workers.dev/https://gowa.megawebs.com/group/participants?device_id=sami&group_id=GROUP_JID"
curl -s "https://cors.trigox.workers.dev/https://gowa.megawebs.com/chat/GROUP_JID/messages?device_id=sami&limit=100"
```

### Workflow D: Bulk Send With WhatsApp Check

```bash
PHONES="34611111111 34622222222 34633333333"
MESSAGE="Your bulk message here"

for phone in $PHONES; do
  check=$(curl -s "https://cors.trigox.workers.dev/https://gowa.megawebs.com/user/check?device_id=sami&phone=$phone")
  if echo "$check" | grep -q "jid"; then
    curl -s -X POST "https://cors.trigox.workers.dev/https://gowa.megawebs.com/send/message" \
      -H "Content-Type: application/json" \
      -d "{\"device_id\":\"sami\",\"phone\":\"${phone}@s.whatsapp.net\",\"message\":\"$MESSAGE\"}"
    echo "Sent to $phone"
    sleep 2
  else
    echo "$phone not on WhatsApp"
  fi
done
```

## Error Handling

| Error | Cause | Fix |
|-------|-------|-----|
| `Cannot GET /...` | Endpoint does not exist | Use the verified endpoints above |
| Empty response | CORS proxy issue | Try the direct URL without the proxy |
| `device not found` | Wrong device ID | Use `device_id=sami` |
| Connection refused | GOWA server down | Wait and retry |
| `not on whatsapp` | Number not registered | Skip or verify the number |

## Quick Reference

| Action | Method | Endpoint | Key Params |
|--------|--------|----------|------------|
| List chats | GET | `/chats` | `device_id`, `limit`, `offset` |
| Read messages | GET | `/chat/{JID}/messages` | `device_id`, `limit`, `offset` |
| Send message | POST | `/send/message` | `device_id`, `phone`, `message` |
| Send text | POST | `/send/text` | `device_id`, `phone`, `message` |
| Check number | GET | `/user/check` | `device_id`, `phone` |
| Group info | GET | `/group/info` | `device_id`, `group_id` |
| Group members | GET | `/group/participants` | `device_id`, `group_id` |

## MCP Alternative

The `Whatsapp Go MCP` server exists but is unreliable. Always use direct `curl` calls to `gowa.megawebs.com` instead.
