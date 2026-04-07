---
name: gowa-whatsapp-api
description: |
  GOWA (Go WhatsApp) REST API for WhatsApp automation via gowa.megawebs.com.
  Triggers: "whatsapp", "gowa", "send whatsapp", "whatsapp message", "whatsapp chats",
  "check whatsapp", "whatsapp group", "group participants", "send message whatsapp",
  "list chats", "read messages", "whatsapp contacts", "whatsapp outreach", "bulk whatsapp",
  "whatsapp check number", "is on whatsapp", "gowa api", "gowa curl", "send link whatsapp",
  "send image whatsapp", "send video whatsapp", "link preview whatsapp".

  Use when: (1) Listing WhatsApp chats/conversations, (2) Reading messages from any chat,
  (3) Sending WhatsApp messages to individuals or groups, (4) Getting group info/participants,
  (5) Checking if a phone number is on WhatsApp, (6) Any WhatsApp automation via REST API,
  (7) Cross-referencing contacts with WhatsApp, (8) Bulk messaging campaigns,
  (9) Sending links with rich previews, (10) Sending images with captions.

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

## Critical Rules

1. ALWAYS use `cors.trigox.workers.dev` proxy for all calls: `curl -s "https://cors.trigox.workers.dev/https://gowa.megawebs.com/..."`
2. NEVER use the WhatsApp Go MCP tools. Always curl directly.
3. `device_id=sami` is required on ALL GET requests as a query parameter.
4. For POST requests, `device_id` goes in the JSON body OR as `X-Device-Id` header.
5. Phone numbers use format `34XXXXXXXXX` (country code + number, no `+` prefix).
6. JID format for individuals: `34XXXXXXXXX@s.whatsapp.net`
7. JID format for groups: `120363XXXXXXXXXXXX@g.us`

---

## Verified Endpoints

### 1. List Chats (GET)

```bash
curl -s "https://cors.trigox.workers.dev/https://gowa.megawebs.com/chats?device_id=sami&limit=100&offset=0"
```

Parameters: `device_id` (required), `limit` (max 100, default 25), `offset` (0-based).

### 2. Read Messages From Chat (GET)

**CRITICAL PATH:** `/chat/{JID}/messages` — JID goes in URL path BEFORE `/messages`.

```bash
curl -s "https://cors.trigox.workers.dev/https://gowa.megawebs.com/chat/34611158350@s.whatsapp.net/messages?device_id=sami&limit=100"
```

- CORRECT: `/chat/{JID}/messages`
- WRONG: `/chat/messages/{JID}`, `/chat/messages?jid={JID}`, `/messages?chat_id={JID}`

### 3. Send Text Message (POST)

```bash
curl -s -X POST "https://cors.trigox.workers.dev/https://gowa.megawebs.com/send/message" \
  -H "Content-Type: application/json" \
  -d '{"device_id":"sami","phone":"34XXXXXXXXX@s.whatsapp.net","message":"Hello","is_forwarded":false}'
```

Alternative: `/send/text` with same params.

### 4. Send Link With Rich Preview (POST) — NEW

Sends a URL with auto-generated thumbnail preview and title. **Use this instead of `/send/message` when sharing links.**

```bash
curl -s -X POST "https://cors.trigox.workers.dev/https://gowa.megawebs.com/send/link" \
  -H "Content-Type: application/json" \
  -d '{"device_id":"sami","phone":"34XXXXXXXXX@s.whatsapp.net","link":"https://example.com/article","caption":"Check this out"}'
```

**Parameters (JSON body):**
- `device_id` (required): `"sami"`
- `phone` (required): JID or plain number
- `link` (required): Full URL to share — NOT `url`, must be `link`
- `caption` (required): Text shown above the preview — NOT `message`, must be `caption`

**IMPORTANT CAVEATS:**
- GOWA fetches the URL server-side to generate preview. If the target site blocks the request (403), the call fails.
- Sites known to BLOCK: La Moncloa (.gob.es), some law firm sites with Cloudflare protection, Parainmigrantes.
- Sites that WORK: Infobae, Instagram, Wikipedia, España Abogados, Familias Solidarias, La Actualidad.
- **FALLBACK:** If `/send/link` returns 403 or INTERNAL_SERVER_ERROR, resend using `/send/message` with the URL in the text body. User will get a clickable link but no preview thumbnail.

**Success response:**
```json
{"code":"SUCCESS","message":"Link sent to 34XXXXXXXXX@s.whatsapp.net (server timestamp: ...)"}
```

**Error responses:**
```json
{"code":"VALIDATION_ERROR","message":"caption: cannot be blank; link: cannot be blank."}
{"code":"INTERNAL_SERVER_ERROR","message":"HTTP request failed with status: 403 Forbidden"}
```

### 5. Send Image (POST) — UNRELIABLE

Endpoint exists but most image URLs fail because GOWA fetches server-side.

```bash
curl -s -X POST "https://cors.trigox.workers.dev/https://gowa.megawebs.com/send/image" \
  -H "Content-Type: application/json" \
  -d '{"device_id":"sami","phone":"34XXXXXXXXX@s.whatsapp.net","image_url":"https://example.com/photo.jpg","caption":"Photo caption"}'
```

Known issues: Most URLs return 403, "unsupported file type" if not direct image MIME.

### 6. Send Video (POST) — UNRELIABLE

Only works with direct video file URLs (.mp4).

```bash
curl -s -X POST "https://cors.trigox.workers.dev/https://gowa.megawebs.com/send/video" \
  -H "Content-Type: application/json" \
  -d '{"device_id":"sami","phone":"34XXXXXXXXX@s.whatsapp.net","video_url":"https://example.com/video.mp4","caption":"Video caption"}'
```

Known issues: Instagram/web page URLs fail with "invalid content type: text/html". For Instagram Reels use `/send/link` instead.

### 7. Check If Phone Is On WhatsApp (GET)

```bash
curl -s "https://cors.trigox.workers.dev/https://gowa.megawebs.com/user/check?device_id=sami&phone=34642609188"
```

### 8. Group Info (GET)

```bash
curl -s "https://cors.trigox.workers.dev/https://gowa.megawebs.com/group/info?device_id=sami&group_id=120363423943239814@g.us"
```

### 9. Group Participants (GET)

```bash
curl -s "https://cors.trigox.workers.dev/https://gowa.megawebs.com/group/participants?device_id=sami&group_id=120363423943239814@g.us"
```

---

## Endpoints That Do NOT Exist

- `/messages?device_id=...&chat_id=...`
- `/message/list?device_id=...`
- `/chat/history?device_id=...&jid=...`
- `/chat/{JID}/history`
- `/chat/messages/{JID}`
- `/chat/messages?jid={JID}`
- `/contacts?device_id=...`
- `/contact/list?device_id=...`
- `/user/contacts?device_id=...`
- `/groups/{JID}?device_id=...`
- `/group/list?device_id=...`

---

## Bulk Send Pattern (Shell-Compatible)

**CRITICAL:** The Claude container uses `/bin/sh`, NOT bash. Bash arrays `()` cause syntax errors. Use function-based pattern:

```bash
send() {
  curl -s -X POST "https://cors.trigox.workers.dev/https://gowa.megawebs.com/send/message" \
    -H "Content-Type: application/json" \
    -d "{\"device_id\":\"sami\",\"phone\":\"34XXXXXXXXX@s.whatsapp.net\",\"message\":\"$1\"}"
  echo " --- $2"
  sleep 1
}

send "First message" "1/10"
send "Second message" "2/10"
```

For links with preview:
```bash
sendlink() {
  curl -s -X POST "https://cors.trigox.workers.dev/https://gowa.megawebs.com/send/link" \
    -H "Content-Type: application/json" \
    -d "{\"device_id\":\"sami\",\"phone\":\"34XXXXXXXXX@s.whatsapp.net\",\"link\":\"$1\",\"caption\":\"$2\"}"
  echo " --- $3"
  sleep 1
}

sendlink "https://example.com/article" "Check this article" "1/5"
```

**Rate limiting:** `sleep 1` between messages is sufficient. Use `sleep 2` for bulk 20+ msgs.

---

## Link Sharing Strategy

| Scenario | Endpoint | Result |
|----------|----------|--------|
| Share link with preview thumbnail | `/send/link` | Rich card with image + title |
| Link to site that blocks bots | `/send/message` | Clickable link, no preview |
| Instagram Reel / video page | `/send/link` | Preview card with play button |
| Direct .mp4 file | `/send/video` | Inline video player |
| Direct .jpg/.png | `/send/image` | Inline image |

**Recommended flow:**
1. Try `/send/link` first for best UX
2. If 403/error, fallback to `/send/message` with URL in text
3. Only use `/send/image` or `/send/video` with known direct-file URLs

---

## Quick Reference

| Action | Method | Endpoint | Key Params |
|--------|--------|----------|------------|
| List chats | GET | `/chats` | `device_id`, `limit`, `offset` |
| Read messages | GET | `/chat/{JID}/messages` | `device_id`, `limit`, `offset` |
| Send text | POST | `/send/message` | `device_id`, `phone`, `message` |
| Send text (alt) | POST | `/send/text` | `device_id`, `phone`, `message` |
| **Send link** | POST | `/send/link` | `device_id`, `phone`, `link`, `caption` |
| Send image | POST | `/send/image` | `device_id`, `phone`, `image_url`, `caption` |
| Send video | POST | `/send/video` | `device_id`, `phone`, `video_url`, `caption` |
| Check number | GET | `/user/check` | `device_id`, `phone` |
| Group info | GET | `/group/info` | `device_id`, `group_id` |
| Group members | GET | `/group/participants` | `device_id`, `group_id` |

## Error Handling

| Error | Cause | Fix |
|-------|-------|-----|
| `Cannot GET /...` | Endpoint does not exist | Use verified endpoints above |
| `VALIDATION_ERROR` | Missing required field | Check param names: `link` not `url`, `caption` not `message` |
| `403 Forbidden` | Target site blocks GOWA fetch | Fallback to `/send/message` |
| `invalid content type: text/html` | URL is a web page not media | Use `/send/link` for web pages |
| `unsupported file type` | URL not valid media | Use direct file URL (.jpg, .mp4) |
| `device not found` | Wrong device_id | Use `device_id=sami` |
| `connection reset by peer` | Target server dropped conn | Retry or fallback to `/send/message` |

## MCP Alternative (NOT RECOMMENDED)

The `Whatsapp Go MCP` server at `gowa-mcp.megawebs.com` exists but is unreliable. Always use direct `curl` to `gowa.megawebs.com`.
