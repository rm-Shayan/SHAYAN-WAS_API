# Evolution API — Integration Guide

Kisi bhi app (Node, Python, PHP, cURL, etc.) se WhatsApp use karne ke liye ye document follow karo.

---

## 0. Connection Info

| Cheez | Value |
|---|---|
| Base URL | `http://localhost:8080` |
| API Key | `4ed9c72c26094827c0472fe08b4a7813527cdc93a5118eb8` |
| Auth header | `apikey: <API_KEY>` |
| Manager UI | `http://localhost:8080/manager` |

**Har request me `apikey` header zaroori hai.** Bina ke `401` aayega.

```bash
curl http://localhost:8080/ \
  -H "apikey: 4ed9c72c26094827c0472fe08b4a7813527cdc93a5118eb8"
```

---

## 1. Instance banao (WhatsApp connect)

Ek "instance" = ek WhatsApp number. Har number ke liye alag instance.

```bash
curl -X POST http://localhost:8080/instance/create \
  -H "apikey: <API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{
    "instanceName": "my-shop",
    "integration": "WHATSAPP-BAILEYS",
    "Integration": "WHATSAPP-BAILEYS",
    "qrcode": true,
    "alwaysOnline": false,
    "readMessages": false,
    "groupsIgnore": false,
    "syncFullHistory": false
  }'
```

### IMPORTANT — is build ki 3 gaddiyan

1. **Body flat hoga**, `{ "instance": {...} }` nested karne se kaam nahi karega.
2. **`integration` (lowercase) bhejo.** Code lowercase padhta hai. Saath me `Integration` (capital) bhi bhej do — validation schema wo check karta hai. Sirf ek bhejoge to error aayega:
   ```json
   { "status": 400, "error": "Bad Request", "response": { "message": ["Invalid integration"] } }
   ```
   Valid values: `WHATSAPP-BAILEYS` | `WHATSAPP-BUSINESS` | `EVOLUTION`
3. **`instanceName` hi asli key hai.** Baaki endpoints me `:instanceName` se call hota hai, ID se nahi.

Response me `instance.id`, `instance.name`, `instance.token` aata hai. **`token` ko save kar le** — instance-specific API calls me `token` header lagta hai.

---

## 2. QR code nikalo aur scan karo

QR **~20-30 second me expire** ho jata hai. Har baar naya le.

```bash
curl http://localhost:8080/instance/connect/my-shop \
  -H "apikey: <API_KEY>"
```

Response:

```json
{
  "pairingCode": "",
  "code": "2@ABC123...",
  "base64": "data:image/png;base64,iVBORw0KG...",
  "count": 0
}
```

- `base64` → ise decode karke QR image banao aur phone se scan karo
- `pairingCode` → 8-digit code, phone me **Link with phone number** se dal sakte ho (fast expire hota hai, ~20 sec)

Node me QR image banana:

```js
const r = await fetch(`http://localhost:8080/instance/connect/${NAME}`, {
  headers: { apikey: KEY },
});
const { base64 } = await r.json();
require('fs').writeFileSync(
  'qrcode.png',
  Buffer.from(base64.replace(/^data:image\/\w+;base64,/, ''), 'base64'),
);
```

**Phone me kahan scan karna hai:**
`WhatsApp → Settings → Linked Devices → Link a Device`

> QR refresh ho raha ho to ek loop chala ke har 15 second me `base64` fetch karte raho. Jab tak user na scan kare, `base64` change hota rahega.

---

## 3. Connection check karo

```bash
curl http://localhost:8080/instance/connectionState/my-shop \
  -H "apikey: <API_KEY>"
```

States: `connecting` → `qr` → `connected` (ya `disconnected`)

Har request pe ye call karke QR loop chalate raho — jab `state === 'connected'` ho jaye, QR band kar do.

Saare instances dekhne ke liye:

```bash
curl http://localhost:8080/instance/fetchInstances -H "apikey: <API_KEY>"
```

---

## 4. Message bhejo

### Text

```bash
curl -X POST http://localhost:8080/message/sendText/my-shop \
  -H "apikey: <API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{
    "number": "919876543210",
    "text": "Hello! Kaise ho?"
  }'
```

**`number` me country code ke saath** hona chahiye, `+` ya `00` ke bina. India ke liye `919876543210`.

Extra options (text message):

| Field | Type | Kaam |
|---|---|---|
| `linkPreview` | boolean | URL ka preview dikhaye ya na |
| `delay` | int (ms) | Send karne se pehle wait |
| `everyOne` | boolean | Sabko `@everyone` mention |
| `mentioned` | string[] | Specific numbers ko mention |
| `quoted` | object | Reply karna |

```json
{
  "number": "919876543210",
  "text": "Dekh raha hai?",
  "mentioned": ["919876543211"],
  "quoted": { "key": { "id": "3EB0ABC123" } }
}
```

### Media (image / audio / video / document)

`multipart/form-data` bhejna hai, JSON nahi:

```bash
curl -X POST http://localhost:8080/message/sendMedia/my-shop \
  -H "apikey: <API_KEY>" \
  -F "number=919876543210" \
  -F "mediatype=image" \
  -F "mimetype=image/png" \
  -F "fileName=photo.png" \
  -F "caption=Yeh dekho" \
  -F "file=@C:/path/photo.png"
```

`mediatype`: `image` | `document` | `video` | `audio`

### Baaki message types

| Endpoint | Kya karta hai |
|---|---|
| `POST /message/sendText/:name` | Text |
| `POST /message/sendMedia/:name` | Image/video/audio/document |
| `POST /message/sendAudio/:name` | Voice note (PTT) |
| `POST /message/sendPtv/:name` | Video (circle, screen record) |
| `POST /message/sendSticker/:name` | Sticker |
| `POST /message/sendStatus/:name` | WhatsApp status |
| `POST /message/sendLocation/:name` | Location |
| `POST /message/sendContact/:name` | Contact card |
| `POST /message/sendPoll/:name` | Poll |
| `POST /message/sendList/:name` | List / broadcast list |
| `POST /message/sendButtons/:name` | Buttons |
| `POST /message/sendReaction/:name` | Emoji reaction |
| `POST /message/sendTemplate/:name` | Approved template message |

---

## 5. Message receive karo (Webhook)

Bhejna to ho gaya, ab aana chahiye. Iske liye webhook set karo.

### Webhook set karo

```bash
curl -X POST http://localhost:8080/webhook/set/my-shop \
  -H "apikey: <API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{
    "enabled": true,
    "url": "https://your-app.com/api/whatsapp",
    "webhookByEvents": false,
    "webhookBase64": false,
    "events": [
      "MESSAGES_UPSERT"
    ]
  }'
```

> `url` public hona chahiye — `localhost` kaam nahi karega. Local test ke liye `ngrok http 3000` use karo.

Webhook me ye events bhejti hain:

| Event | Kab |
|---|---|
| `MESSAGES_UPSERT` | Naya message aaya (**ye main hai**) |
| `MESSAGES_SET` | Poora chat history sync hua |
| `SEND_MESSAGE` | Tumne message bheja |
| `MESSAGES_UPDATE` | Message status badla (delivered/read) |
| `CONNECTION_UPDATE` | WhatsApp connect/disconnect hua |
| `QRCODE_UPDATED` | QR refresh hua |
| `CONTACTS_UPSERT` | Naya contact |

### Webhook receiver (Node/Express)

```js
app.post('/api/whatsapp', (req, res) => {
  const { event, instance, data } = req.body;

  // Turant 200 bhejo warna Evolution retry karega
  res.sendStatus(200);

  if (event === 'messages.upsert') {
    for (const msg of data.messages) {
      if (msg.key.fromMe) continue; // apne bheje hue msg ignore

      const from = msg.key.remoteJid.split('@')[0];
      const text = msg.message?.conversation || msg.message?.extendedTextMessage?.text;

      console.log(`${from}: ${text}`);
      // yahan apna logic: auto-reply, save to DB, agent ko bhejo
    }
  }

  if (event === 'connection.update') {
    console.log('WhatsApp state:', data.state);
  }
});
```

**Webhook handler hamesha turant `200` return karo.** Late response ya error me Evolution 10 baar retry karta hai.

---

## 6. WhatsApp number chahiye to number verify karo

Apna linked number check karne ke liye (WhatsApp Business API me zaroori):

```bash
curl -X POST http://localhost:8080/chat/whatsappNumbers/my-shop \
  -H "apikey: <API_KEY>"
```

---

## 7. Instance manage karo

| Kaam | Call |
|---|---|
| Restart | `POST /instance/restart/:name` |
| Connection state | `GET /instance/connectionState/:name` |
| WhatsApp se logout | `DELETE /instance/logout/:name` |
| Delete (poora) | `DELETE /instance/delete/:name` |
| Presence set | `POST /instance/setPresence/:name` |

---

## 8. Poora Node.js example (QR + auto-reply bot)

```js
const KEY = '4ed9c72c26094827c0472fe08b4a7813527cdc93a5118eb8';
const BASE = 'http://localhost:8080';
const NAME = 'my-shop';
const fs = require('fs');

const api = (method, path, body) =>
  fetch(BASE + path, {
    method,
    headers: { apikey: KEY, 'Content-Type': 'application/json' },
    body: body ? JSON.stringify(body) : undefined,
  }).then((r) => r.json());

async function connect() {
  // 1. Instance banao (agar pehle se hai to skip)
  const existing = await api('GET', '/instance/fetchInstances');
  if (!existing.find((i) => i.name === NAME)) {
    await api('POST', '/instance/create', {
      instanceName: NAME,
      integration: 'WHATSAPP-BAILEYS',
      Integration: 'WHATSAPP-BAILEYS',
      qrcode: true,
    });
  }

  // 2. QR loop — jab tak connect na ho, QR refresh hota rahega
  while (true) {
    const state = await api('GET', `/instance/connectionState/${NAME}`);
    if (state.state === 'connected') {
      console.log('WhatsApp connected!');
      break;
    }

    const qr = await api('GET', `/instance/connect/${NAME}`);
    if (qr.base64) {
      fs.writeFileSync(
        'qrcode.png',
        Buffer.from(qr.base64.replace(/^data:image\/\w+;base64,/, ''), 'base64'),
      );
      console.log('QR updated -> qrcode.png (scan karo)');
    }
    await new Promise((r) => setTimeout(r, 15000));
  }

  // 3. Webhook set karo
  await api('POST', `/webhook/set/${NAME}`, {
    enabled: true,
    url: 'https://your-app.com/api/whatsapp',
    events: ['MESSAGES_UPSERT', 'CONNECTION_UPDATE'],
  });
}

// Auto-reply bot
async function onWebhook(body) {
  if (body.event === 'messages.upsert') {
    for (const msg of body.data.messages) {
      if (msg.key.fromMe) continue;
      const from = msg.key.remoteJid.split('@')[0];
      const text = msg.message?.conversation;
      if (!text) continue;

      if (text.toLowerCase() === 'ping') {
        await api('POST', `/message/sendText/${NAME}`, {
          number: from,
          text: 'pong',
        });
      }
    }
  }
}

module.exports = { connect, onWebhook };
```

---

## 9. Python example

```python
import requests, time, base64

BASE  = "http://localhost:8080"
KEY   = "4ed9c72c26094827c0472fe08b4a7813527cdc93a5118eb8"
NAME  = "my-shop"
H     = {"apikey": KEY, "Content-Type": "application/json"}

requests.post(f"{BASE}/instance/create", headers=H, json={
    "instanceName": NAME,
    "integration": "WHATSAPP-BAILEYS",
    "Integration": "WHATSAPP-BAILEYS",
    "qrcode": True,
})

while True:
    s = requests.get(f"{BASE}/instance/connectionState/{NAME}", headers=H).json()
    if s.get("state") == "connected":
        print("connected!")
        break
    qr = requests.get(f"{BASE}/instance/connect/{NAME}", headers=H).json()
    if qr.get("base64"):
        data = qr["base64"].split(",", 1)[1]
        open("qrcode.png", "wb").write(base64.b64decode(data))
        print("QR updated -> qrcode.png")
    time.sleep(15)

# message bhejo
requests.post(f"{BASE}/message/sendText/{NAME}", headers=H, json={
    "number": "919876543210",
    "text": "Hello from Python!",
}).print()
```

---

## 10. Troubleshooting

| Problem | Reason / Fix |
|---|---|
| `Invalid integration` | `integration` lowercase bhejo, aur saath me `Integration` bhi |
| 404 on `/instance/connect/...` | ID nahi, **name** use karo: `/instance/connect/my-shop` |
| QR scan ke baad bhi `connecting` | Instance ke naam me `-` ya `_` mat rakho |
| QR expire ho gaya | Naya `/instance/connect` call karo, QR ~20-30 sec me expire hota hai |
| Webhook call nahi aa raha | URL public hona chahiye — `localhost` kaam nahi karega, `ngrok` use karo |
| Webhook double hit ho raha | `200` turant return karo warna retry hoga (10 baar) |
| `too many connections` | Supabase free tier chhota hai — `.env` me `connection_limit=5` rakha hai |
| Redis `disconnected` log | Redis Cloud free tier idle timeout — normal, auto-reconnect hota hai |
| Server restart ke baad instance gayab | Nahi hoga — DB me saved hai; `fetchInstances` se wapas aa jayega |

---

## 11. Security note

- API key abhi chat me share hui hai — production me jaane se pehle `.env` me badal do
- Instance `token` bhi sensitive hai, kabhi public repo me mat daalo
- `.env` file `.gitignore` me hai, kabhi commit mat karo

---

## Quick reference

```
POST   /instance/create                          # instance banao
GET    /instance/connect/:name                   # QR code lo
GET    /instance/connectionState/:name           # status check
GET    /instance/fetchInstances                  # sabhi instances
DELETE /instance/delete/:name                    # instance delete

POST   /message/sendText/:name                   # text bhejo
POST   /message/sendMedia/:name                  # media bhejo (multipart)
POST   /message/sendLocation/:name
POST   /message/sendContact/:name
POST   /message/sendPoll/:name

POST   /webhook/set/:name                        # webhook lagao
GET    /webhook/find/:name                       # webhook dekho

POST   /chat/whatsappNumbers/:name               # linked number verify
```