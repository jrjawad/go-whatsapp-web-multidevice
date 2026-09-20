# MCP (Model Context Protocol) Guide (اردو)

Yeh guide batati hai ke apne Docker container mein chal rahe GOWA server ka **MCP server**
kaise use karein — AI agents (Claude Desktop, Claude Code, Cursor, waghera) ko WhatsApp
control dene ke liye. Detailed OAuth setup ke liye [`docs/mcp-oauth.md`](../docs/mcp-oauth.md)
dekhein.

## 1. MCP kya hai is project mein

MCP ek **alag process nahi hai** — jab bhi `rest` mode chal raha ho (jo Docker image
default run karta hai), MCP endpoint khud-ba-khud available hota hai:

```
http://<host>:<port><base-path>/mcp
```

Default: `http://localhost:3000/mcp`

- Yeh REST server jaisa hi auth, jaisa hi device manager, aur jaisa hi data use karta hai.
- Agar disable karna ho: `--mcp-enabled=false` ya `MCP_ENABLED=false` (default: enabled).

## 2. Confirm karein MCP enabled hai

Container ke logs check karein ya seedha curl se test karein:

```bash
curl -i -X POST http://localhost:3000/mcp
```

Agar Basic Auth set hai to `401` milega (normal hai — credentials ke bina). Agar
`MCP_ENABLED=false` kiya ho to route hi nahi milega (404).

## 3. Available MCP tools

40+ granular tools ki bajaye, **5 consolidated tools** hain — har tool ek `type`/`action`
parameter leta hai jo decide karta hai konsa operation karna hai:

| Tool               | `type` / `action` values                                                                                                                                                     |
|---------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `whatsapp_send`    | `text`, `image`, `video`, `audio`, `document`, `sticker`, `location`, `contact`, `poll`, `link`, `forward`                                                                     |
| `whatsapp_message` | `react`, `edit`, `revoke`, `delete`, `mark_read`, `mark_played`, `star`, `unstar`, `download_media`                                                                            |
| `whatsapp_chat`    | `list_chats`, `list_contacts`, `get_messages`, `archive`                                                                                                                       |
| `whatsapp_group`   | `create`, `join_with_link`, `leave`, `info`, `participants`, `add_participants`, `remove_participants`, `promote`, `demote`, `invite_link`, `set_name`, `set_topic`, `set_settings`, `join_requests`, `manage_join_requests` |
| `whatsapp_app`     | `status`, `login_qr`, `login_code`, `logout`, `reconnect`                                                                                                                      |

Yani agar AI agent ko "yeh image bhejo" bolna hai, wo `whatsapp_send` tool ko
`type: "image"` ke sath call karega — alag se `send_image` tool nahi hai.

## 4. Multi-device selection

Agar aap ke paas ek se zyada WhatsApp number connected hain:

- MCP client connection par `X-Device-Id` header set karein — us connection ke tamam
  tool calls usi device par apply honge.
- Agar header nahi diya to default/sole device use hoga (REST jaisa hi rule).
- Kisi individual call mein alag device chahiye ho to `device_id` argument bhi pass kar
  sakte hain — wo header ko override karega.

## 5. Claude Desktop / Claude Code se connect karna

`claude_desktop_config.json` (ya jo bhi MCP-client config file ho) mein:

```json
{
  "mcpServers": {
    "whatsapp": {
      "url": "http://localhost:3000/mcp",
      "headers": {
        "Authorization": "Basic dXNlcjpzZWNyZXQ=",
        "X-Device-Id": "628123456789"
      }
    }
  }
}
```

- `Authorization` sirf tab chahiye jab server par `--basic-auth` / `APP_BASIC_AUTH` set ho.
  Value `base64(username:password)` honi chahiye (`echo -n "admin:admin" | base64`).
- `X-Device-Id` sirf multi-device setup mein chahiye.
- `headers` block optional hai — single device + auth-disabled setup mein sirf `url` kafi hai.

Agar client Basic Auth header attach nahi kar sakta (kuch remote/custom connectors),
tab OAuth 2.1 use karein — neeche section 7 dekhein.

## 6. Cursor / other MCP clients

Zyada tar MCP clients ek jaisi hi config lete hain — server type `http`/`streamable-http`,
`url` = `http://localhost:3000/mcp`, aur agar auth chahiye to custom header. Client ki apni
docs mein "Authorization" ya "custom headers" field dekhein.

**Note:** MCP endpoint sirf POST/DELETE accept karta hai (SSE/idle GET stream jaan-bujh
kar disable hai taake ek khula GET connection resources na roke rakhe) — ye normal hai,
sirf standard streamable-HTTP MCP client hi is se baat kar payenge.

## 7. Remote clients ke liye OAuth (jab Basic Auth attach nahi ho sakta)

Kuch remote MCP clients (jaise Claude ka custom connector feature) direct Authorization
header set nahi kar sakte — un ke liye GOWA khud ek OAuth 2.1 server bhi bana sakta hai.

Yeh **disable by default** hai. Enable karne ke liye (public HTTPS domain zaroori hai):

```env
APP_BASIC_AUTH=admin:replace-with-a-strong-password
MCP_ENABLED=true
MCP_OAUTH_ENABLED=true
MCP_OAUTH_ISSUER_URL=https://gowa.example.com
```

- `MCP_OAUTH_ISSUER_URL` public HTTPS URL honi chahiye (localhost/internal Docker
  hostname nahi — jo bhi URL client se pahunch sakta ho, wahi).
- Enable hone ke baad `/mcp` `Authorization: Bearer <token>` **ya** existing Basic Auth
  dono accept karta hai.
- Sign-in wahi `APP_BASIC_AUTH` credentials se hota hai — koi alag user database nahi.
- Full details, reverse-proxy requirements, `APP_BASE_PATH` ke sath setup, aur security
  model: [`docs/mcp-oauth.md`](../docs/mcp-oauth.md).

## 8. Docker mein MCP env vars set karna

`docker-compose.yml` (aap ka container jo already chal raha hai, usi mein add karein):

```yaml
services:
  whatsapp:
    image: aldinokemal2104/go-whatsapp-web-multidevice
    container_name: whatsapp
    restart: always
    ports:
      - "3000:3000"
    volumes:
      - whatsapp_storages:/app/storages
      - whatsapp_statics:/app/statics
    command:
      - rest
      - --basic-auth=admin:admin
      - --mcp-enabled=true

volumes:
  whatsapp_storages:
  whatsapp_statics:
```

Change ke baad:

```bash
docker compose up -d
```

## 9. Troubleshooting

| Masla | Wajah / Hal |
|---|---|
| `/mcp` par 404 | `MCP_ENABLED=false` hai — env/flag check karein aur container restart karein. |
| `/mcp` par 401 | Basic Auth set hai lekin client ne header nahi bheja, ya galat credentials. |
| Har tool call galat device par ja raha hai | `X-Device-Id` header ya `device_id` argument confirm karein; ek se zyada device registered hone par yeh zaroori hai. |
| Remote client connect nahi ho raha (SSE ki baat kar raha hai) | Client ko streamable-HTTP MCP support chahiye, purana SSE-only client kaam nahi karega. |
| Claude custom connector Basic Auth nahi bhej pa raha | Section 7 (OAuth) follow karein. |

## Related

- [`readme.md`](../readme.md) → "MCP Server (Model Context Protocol)" section — yehi info English mein.
- [`docs/mcp-oauth.md`](../docs/mcp-oauth.md) — OAuth deep-dive.
- Postman collection app/device folders — REST equivalents jo MCP tools internally call
  karte hain, agar aap direct REST se bhi test karna chahein.
