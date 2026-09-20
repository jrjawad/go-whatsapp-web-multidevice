# myDocs — GOWA API, Webhook & MCP Guide (اردو)

Yeh folder aap ke liye teen cheezein rakhta hai: mukammal Postman collection, aur webhook +
MCP use karne ke Urdu guides. Yeh sab is repo ke apne `docs/openapi.yaml` /
`docs/webhook-payload.md` / `docs/mcp-oauth.md` se derive kiye gaye hain — wahi files
"source of truth" hain, yahan sirf quick-start aur ready-to-use collection hai.

## Is folder mein kya hai

| File | Kya hai |
|---|---|
| [`GOWA-API.postman_collection.json`](./GOWA-API.postman_collection.json) | Mukammal Postman collection — 85 requests, 10 folders (app, device, user, send, message, call, chat, group, newsletter, chatwoot), sab `docs/openapi.yaml` se generate + real example values ke sath. |
| [`GOWA-Docker.postman_environment.json`](./GOWA-Docker.postman_environment.json) | Postman environment — `baseUrl`, auth, `deviceId`, example JIDs. |
| [`webhook-guide.md`](./webhook-guide.md) | Webhook enable/configure karne, payload samajhne, aur signature verify karne ka tariqa. |
| [`mcp-guide.md`](./mcp-guide.md) | MCP server (AI agent integration) use karne ka tariqa — Claude/Cursor config, tools list, OAuth. |

## 1. Postman mein collection import karna

1. Postman kholein → **Import** button (top-left).
2. Dono files drag-drop karein (ya "Choose Files"):
   - `GOWA-API.postman_collection.json`
   - `GOWA-Docker.postman_environment.json`
3. Import hone ke baad, top-right environment dropdown se **"GOWA - Docker (localhost)"**
   select karein.
4. Environment ki values edit karein (environment ka pencil/eye icon):
   - `baseUrl` — agar container ka port `3000` ke ilawa kuch aur hai to change karein
     (e.g. `http://localhost:8080`).
   - `basicAuthUsername` / `basicAuthPassword` — sirf tab bharein jab container
     `--basic-auth` / `APP_BASIC_AUTH` ke sath chal raha ho.
   - `deviceId` — agar ek se zyada WhatsApp device connected hain to `GET /app/devices`
     ya `GET /devices` call karke wahan se value copy karein.

## 2. Pehli call test karna

Sab se pehle **health check** try karein (auth ki zaroorat nahi):

```
GET {{baseUrl}}/health
```

Phir login/connection status dekhein:

```
GET {{baseUrl}}/app/status
```

Agar `basicAuthUsername`/`basicAuthPassword` set kiye hain, Postman collection-level
Basic Auth khud attach kar dega (collection ka **Authorization** tab already
`{{basicAuthUsername}}` / `{{basicAuthPassword}}` par set hai).

## 3. Collection ka structure (folders = API tags)

- **app** — health check, login (QR/pairing-code/passkey), logout, reconnect, devices list, status, server info.
- **device** — multi-device management: add/list/remove device, per-device login/logout/reconnect/status, per-device webhook config.
- **user** — user info, avatar, push name, privacy settings, my groups/newsletters/contacts, WhatsApp check, business profile.
- **send** — message, image, audio, file, sticker, video, contact, link, location, poll, presence, chat-presence (typing indicator).
- **message** — revoke, delete, react, edit, mark as read/played, star/unstar, forward, download media.
- **call** — reject an incoming call.
- **chat** — list chats, get messages, request older history, pin/archive/disappearing-timer.
- **group** — create, participants (add/remove/promote/demote/export), invite links, join requests, name/topic/photo/settings.
- **newsletter** — unfollow, list channel messages, download channel media.
- **chatwoot** — sync, per-device Chatwoot config, webhook endpoints (see [`docs/chatwoot.md`](../docs/chatwoot.md) for full setup).

Har request mein:

- Zaroori headers already lagay hain (`X-Device-Id` jahan chahiye, multipart requests ke
  liye `Content-Type` jaan-bujh kar chhora gaya hai taake Postman khud boundary set kare).
- JSON/form bodies mein **real example values** hain (`docs/openapi.yaml` ke examples se),
  sirf apni real values (phone number, chat JID, message ID) daal kar bhej dein.
- File-upload fields (image/video/audio/document/sticker) khaali hain — Postman mein
  us field par click karke apni file select karein.

## 4. Webhook aur MCP guides

- Webhook setup, payload shape, signature verification: **[webhook-guide.md](./webhook-guide.md)**
- MCP server (Claude/Cursor jaise AI agents ke liye) setup aur tools list: **[mcp-guide.md](./mcp-guide.md)**

## 5. Agar collection dobara generate karni ho (optional, advanced)

Agar future mein `docs/openapi.yaml` update ho (naye endpoints add hon), collection ko
dobara generate karne ke liye (Node.js/npx zaroori hai):

```bash
npx openapi-to-postmanv2 -s docs/openapi.yaml -o myDocs/GOWA-API.postman_collection.json -p -O folderStrategy=Tags
```

Is se placeholder values (`<string>`) wapis aa jayengi — real examples wapis chahiye hon
to project maintainer/Claude se dobara enrichment script chalwayein.
