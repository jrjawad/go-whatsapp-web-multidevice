# Webhook Guide (اردو)

Yeh guide batati hai ke apne Docker mein chal rahe GOWA server se **webhooks** kaise
enable/configure karein, payload ka structure kya hai, aur signature verify kaise karein.
Full/authoritative reference hamesha [`docs/webhook-payload.md`](../docs/webhook-payload.md)
hai — yeh guide usi ka Urdu/Roman-Urdu quick-start hai.

## 1. Webhook kya karta hai

Jab bhi WhatsApp par koi event aata hai (naya message, reaction, delivery/read receipt,
group change, waghera), GOWA us event ka JSON banake aap ke diye gaye URL par
**HTTP POST** kar deta hai. Aapko koi polling nahi karni — server khud push karega.

## 2. Webhook enable karna (Docker)

Aap ka container already chal raha hai, is liye sirf iske start command/compose file mein
flags ya env vars add karne hain, phir container restart karna hai.

### Docker run / CLI flags

```bash
docker run --detach \
  --publish 3000:3000 \
  --name whatsapp \
  --restart always \
  --volume whatsapp-storages:/app/storages \
  --volume whatsapp-statics:/app/statics \
  aldinokemal2104/go-whatsapp-web-multidevice \
  rest --webhook="https://yourapp.com/webhook" --webhook-secret="super-secret-key"
```

### docker-compose.yml

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
      - --webhook=https://yourapp.com/webhook
      - --webhook-secret=super-secret-key
      - --webhook-events=message,message.ack,group.participants

volumes:
  whatsapp_storages:
  whatsapp_statics:
```

Ya `.env` / environment variables se (`docker-compose.yml` mein `environment:` block ya
`--env-file .env`):

```env
WHATSAPP_WEBHOOK=https://yourapp.com/webhook
WHATSAPP_WEBHOOK_SECRET=super-secret-key
WHATSAPP_WEBHOOK_EVENTS=message,message.ack,group.participants
```

Change karne ke baad container restart karein:

```bash
docker compose up -d          # docker-compose use kar rahe hain to
# ya
docker restart whatsapp
```

Multiple webhook URLs bhi comma-separated de sakte hain (`--webhook="https://a.com,https://b.com"`).

## 3. Per-device webhook (multi-device setups)

Agar aap ke paas ek se zyada WhatsApp devices connected hain, to har device ka apna
alag webhook URL/secret set kiya ja sakta hai — bina container restart kiye, seedha REST
API se:

```bash
curl -X PATCH "http://localhost:3000/devices/<device_id>/webhook" \
  -u admin:admin \
  -H "Content-Type: application/json" \
  -d '{"webhook_url": "https://device-specific.example.com/handler"}'
```

- Jab device ka apna `webhook_url` set ho, us device ke events sirf usi URL par jayenge
  (global `--webhook` par nahi) — jab tak `WHATSAPP_WEBHOOK_DEVICE_MERGE_GLOBAL=true`
  na kiya ho (tab dono ko milta hai).
- `webhook_url` ko empty string (`""`) set karke wapis global webhook par la sakte hain.
- Yeh sab endpoints Postman collection ke **device** folder mein maujood hain
  (`GET/PATCH /devices/:device_id/webhook`).

## 4. Kaun se events milte hain — filtering

Default (kuch bhi set na kiya) mein **sab events** forward hote hain. Sirf specific events
chahiye to `--webhook-events` / `WHATSAPP_WEBHOOK_EVENTS` use karein (comma-separated):

| Event                 | Kab fire hota hai                                   |
|------------------------|------------------------------------------------------|
| `message`              | Text, media, contact, location messages              |
| `message.reaction`     | Emoji reaction kisi message par                      |
| `message.revoked`      | Message delete-for-everyone                          |
| `message.edited`       | Message edit                                         |
| `message.ack`          | Delivery/read receipt                                |
| `message.deleted`      | Aapki taraf se message delete                        |
| `chat_presence`        | Typing / recording indicator                         |
| `group.participants`   | Group join/leave/promote/demote                      |
| `group.joined`         | Aapko group mein add kiya gaya                       |
| `label.edit`           | WhatsApp label metadata change                       |
| `label.association`    | Chat par label lagaya/hataya gaya                    |
| `newsletter.joined`    | Channel subscribe                                    |
| `newsletter.left`      | Channel unsubscribe                                  |
| `newsletter.message`   | Channel mein naya message                            |
| `newsletter.mute`      | Channel mute setting change                          |
| `call.offer`           | Incoming call                                        |

```env
# Sirf message aur read receipt chahiye
WHATSAPP_WEBHOOK_EVENTS=message,message.ack
```

Kisi chat/sender ko mute karna ho (event type filter se alag, JID-based filter):

```env
# Sab groups mute
WHATSAPP_WEBHOOK_IGNORE_JIDS=@g.us
# Groups + ek specific 1:1 chat mute
WHATSAPP_WEBHOOK_IGNORE_JIDS=@g.us,628123456789@s.whatsapp.net
```

## 5. Payload ka shape

Har webhook POST body ka top-level structure hamesha yeh hota hai:

```json
{
  "event": "message",
  "device_id": "628987654321@s.whatsapp.net",
  "session_id": "org_2",
  "payload": { }
}
```

- `event` — event ka naam (upar wali table se).
- `device_id` — us device ki JID jise event mila (multi-device setup mein zaroori).
- `session_id` — agar device add karte waqt aapne koi session id di thi, wo yahan milegi.
- `payload` — event-specific data (message body, sender, chat_id, waghera).

Example — sada text message:

```json
{
  "event": "message",
  "device_id": "628987654321@s.whatsapp.net",
  "payload": {
    "id": "3EB0C127D7BACC83D6A1",
    "chat_id": "628987654321@s.whatsapp.net",
    "from": "628123456789@s.whatsapp.net",
    "sender_display_name": "Saved Contact",
    "from_name": "John Doe",
    "timestamp": "2023-10-15T10:30:00Z",
    "is_from_me": false,
    "body": "Hello, how are you?"
  }
}
```

Har event type (reply, reaction, media, group change, call, waghera) ke full examples
[`docs/webhook-payload.md`](../docs/webhook-payload.md) mein maujood hain — jab
apna receiver code likhein to us file ko dekh kar exact fields confirm kar lein.

## 6. Signature verify karna (security)

Har webhook request ke saath ek HMAC-SHA256 signature header aata hai:

- Header: `X-Hub-Signature-256`
- Format: `sha256=<hex digest>`
- Key: aap ka `--webhook-secret` / `WHATSAPP_WEBHOOK_SECRET` (default `secret` — production
  mein zaroor apna secret set karein).

### Node.js example

```javascript
const crypto = require('crypto');

function verifyWebhookSignature(rawBody, signatureHeader, secret) {
  const expected = crypto
    .createHmac('sha256', secret)
    .update(rawBody, 'utf8')
    .digest('hex');

  const received = signatureHeader.replace('sha256=', '');
  return crypto.timingSafeEqual(Buffer.from(expected, 'hex'), Buffer.from(received, 'hex'));
}

app.post('/webhook', express.raw({ type: '*/*' }), (req, res) => {
  const ok = verifyWebhookSignature(req.body, req.headers['x-hub-signature-256'], process.env.WEBHOOK_SECRET);
  if (!ok) return res.status(401).send('invalid signature');

  const event = JSON.parse(req.body.toString('utf8'));
  console.log(event.event, event.payload);
  res.sendStatus(200);
});
```

### Python example

```python
import hmac
import hashlib

def verify_webhook_signature(raw_body: bytes, signature_header: str, secret: str) -> bool:
    expected = hmac.new(secret.encode("utf-8"), raw_body, hashlib.sha256).hexdigest()
    received = signature_header.replace("sha256=", "")
    return hmac.compare_digest(expected, received)
```

**Important:** signature raw request body (bytes) par calculate hoti hai, isliye
JSON-parse karne se **pehle** verify karein — parsed/re-serialized JSON par HMAC match
nahi karega.

## 7. TLS / self-signed certificate issue

Agar aap ka webhook endpoint self-signed cert ya Cloudflare tunnel ke peeche hai aur
`x509: certificate signed by unknown authority` error aa raha hai:

```bash
--webhook-insecure-skip-verify=true
# ya
WHATSAPP_WEBHOOK_INSECURE_SKIP_VERIFY=true
```

Sirf development/testing mein use karein — production mein proper TLS cert use karein.

## 8. Quick test — local receiver

Testing ke liye ek quick local receiver:

```bash
# webhook.site jaisi free service use karke turant dekh sakte hain, ya:
python3 -c "
from http.server import BaseHTTPRequestHandler, HTTPServer
class H(BaseHTTPRequestHandler):
    def do_POST(self):
        length = int(self.headers['Content-Length'])
        print(self.headers.get('X-Hub-Signature-256'))
        print(self.rfile.read(length).decode())
        self.send_response(200)
        self.end_headers()
HTTPServer(('0.0.0.0', 8000), H).serve_forever()
"
```

Phir `--webhook="http://<your-machine-ip>:8000"` set karke ek message bhejein aur
terminal mein payload print hote dekhein.

## Related

- Reference: [`docs/webhook-payload.md`](../docs/webhook-payload.md) — sab event types
  ke full JSON examples, group/newsletter/call payloads, `sender_display_name`
  resolution ka logic.
- Postman collection (`myDocs/GOWA-API.postman_collection.json`) → **device** folder →
  `GET/PATCH /devices/:device_id/webhook` — per-device webhook manage karne ke liye.
