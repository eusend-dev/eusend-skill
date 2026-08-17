# Webhooks

Real-time HTTP POSTs when an email's state changes. Deliveries are retried up to 3 times
with exponential backoff. Your endpoint must return `2xx` **directly** — a `3xx` redirect is
not followed and counts as a failure.

## Subscribe

```ts
const { data } = await client.webhooks.create({
  url: 'https://yourapp.com/webhooks/eusend',
  events: ['email.delivered', 'email.bounced', 'email.complained'],
  // or: events: ['*'] for everything, including future event types
})
data.secret // store this now — needed to verify signatures
```

Lost the secret? `GET /webhooks/:id/secret`, or the dashboard. Rotating with
`POST /webhooks/:id/rotate-secret` takes effect **immediately** — the old secret stops
verifying on the next delivery, so update your endpoint first unless you're rotating
because it leaked.

The URL must be publicly reachable. Private, loopback, and internal addresses are rejected
at creation _and_ re-checked after DNS resolution before every delivery, so pointing a
webhook at an internal host won't work even if DNS changes later.

## Events

| Event              | Fires when                                                                                                                |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------- |
| `email.sent`       | Accepted and in transit. Adds `provider_message_id`                                                                       |
| `email.delivered`  | Receiving server confirmed delivery. Adds `recipients`                                                                    |
| `email.bounced`    | Hard bounce; the address was auto-suppressed. Adds `bounce_type`, `diagnostic`, `smtp_code` or `dsn_status`, `recipients` |
| `email.complained` | Marked as spam; auto-suppressed. Adds `recipients`                                                                        |
| `email.opened`     | Opened — requires `track_opens`. Fires once per email                                                                     |
| `email.clicked`    | Tracked link clicked — requires `track_clicks`. Adds `link_id`, `url`                                                     |

Soft/transient failures are retried internally and never fire `email.bounced`.

## Payload

```json
{
  "type": "email.delivered",
  "email_id": "9a8b7c6d-5e4f-4a3b-8c1d-0e9f8a7b6c5d",
  "recipients": ["alice@example.com"],
  "tags": { "category": "shipping_update" },
  "timestamp": "2026-05-20T10:00:03.000Z"
}
```

`type`, `email_id`, `tags`, and `timestamp` are always present. `tags` is `{}` when the send
had none — never absent — so `payload.tags.category` is safe to read without guarding. Test
mode deliveries also carry `test_mode: true`.

## Verification

Headers on every delivery:

| Header              | Contents                   |
| ------------------- | -------------------------- |
| `webhook-id`        | Unique delivery ID         |
| `webhook-timestamp` | Unix seconds               |
| `webhook-signature` | `v1,` + base64 HMAC-SHA256 |

The signature is `HMAC-SHA256(secret, "{webhook-id}.{webhook-timestamp}.{raw body}")`,
base64-encoded, prefixed with `v1,`. Svix-compatible.

**Two things break this, both silently:**

1. **Parsing JSON before verifying.** The signature covers the exact bytes received.
   `JSON.stringify(JSON.parse(body))` is a different byte string. Capture the raw body.
2. **Non-constant-time comparison.** `===` leaks how much of the signature matched.

### Node (Express)

`express.json()` discards the raw body — use `express.raw()` on this route.

```ts
import express from 'express'
import { createHmac, timingSafeEqual } from 'node:crypto'

app.post('/webhooks/eusend', express.raw({ type: 'application/json' }), (req, res) => {
  const body = req.body.toString('utf8')
  const signed = `${req.headers['webhook-id']}.${req.headers['webhook-timestamp']}.${body}`
  const expected =
    'v1,' + createHmac('sha256', process.env.EUSEND_WEBHOOK_SECRET!).update(signed).digest('base64')

  const received = Buffer.from(String(req.headers['webhook-signature'] ?? ''), 'utf8')
  const digest = Buffer.from(expected, 'utf8')

  // timingSafeEqual throws on a length mismatch — check length first, or a
  // missing/truncated header crashes the handler instead of returning 401.
  if (received.length !== digest.length || !timingSafeEqual(received, digest)) {
    return res.sendStatus(401)
  }

  const event = JSON.parse(body)
  // Acknowledge fast; do the work out of band.
  res.sendStatus(200)
  void handleEvent(event)
})
```

### Node (Next.js route handler / Web standard)

```ts
export async function POST(req: Request) {
  const body = await req.text() // raw, before any parsing
  const signed = `${req.headers.get('webhook-id')}.${req.headers.get('webhook-timestamp')}.${body}`
  const expected =
    'v1,' + createHmac('sha256', process.env.EUSEND_WEBHOOK_SECRET!).update(signed).digest('base64')

  const received = Buffer.from(req.headers.get('webhook-signature') ?? '', 'utf8')
  const digest = Buffer.from(expected, 'utf8')
  if (received.length !== digest.length || !timingSafeEqual(received, digest)) {
    return new Response('invalid signature', { status: 401 })
  }

  await handleEvent(JSON.parse(body))
  return new Response(null, { status: 200 })
}
```

### Python (Flask)

```python
import base64, hashlib, hmac, os
from flask import request, abort

@app.post('/webhooks/eusend')
def eusend_webhook():
    body = request.get_data()  # bytes, unparsed
    signed = b'%s.%s.%s' % (
        request.headers.get('webhook-id', '').encode(),
        request.headers.get('webhook-timestamp', '').encode(),
        body,
    )
    mac = hmac.new(os.environ['EUSEND_WEBHOOK_SECRET'].encode(), signed, hashlib.sha256)
    expected = 'v1,' + base64.b64encode(mac.digest()).decode()

    if not hmac.compare_digest(expected, request.headers.get('webhook-signature', '')):
        abort(401)

    handle_event(request.get_json())
    return '', 200
```

### Go

```go
func handler(w http.ResponseWriter, r *http.Request) {
	body, err := io.ReadAll(r.Body)
	if err != nil {
		w.WriteHeader(http.StatusBadRequest)
		return
	}

	signed := fmt.Sprintf("%s.%s.%s",
		r.Header.Get("webhook-id"), r.Header.Get("webhook-timestamp"), body)

	mac := hmac.New(sha256.New, []byte(os.Getenv("EUSEND_WEBHOOK_SECRET")))
	mac.Write([]byte(signed))
	expected := "v1," + base64.StdEncoding.EncodeToString(mac.Sum(nil))

	if !hmac.Equal([]byte(expected), []byte(r.Header.Get("webhook-signature"))) {
		w.WriteHeader(http.StatusUnauthorized)
		return
	}

	handleEvent(body)
	w.WriteHeader(http.StatusOK)
}
```

## Handler design

**Acknowledge first, work after.** Return `2xx` as soon as the signature checks out and
process asynchronously. A handler that does slow work inline times out and gets retried,
which is how one event becomes four.

**Be idempotent.** Retries mean the same `webhook-id` can arrive more than once. Key your
processing on it.

**Route on tags, not lookups.** Every event carries the send's tags, so one endpoint serves
several kinds of mail without fetching the email first:

```ts
if (event.type === 'email.bounced' && event.tags.category === 'password_reset') {
  // a reset mail that bounced is a locked-out user
  alertOnCriticalBounce(event)
}
```

## Reading a bounce

```json
{
  "type": "email.bounced",
  "email_id": "9a8b7c6d-…",
  "bounce_type": "Uncategorized",
  "smtp_code": 554,
  "diagnostic": "Your access to this mail system has been rejected due to poor reputation…",
  "recipients": ["alice@example.com"],
  "tags": {},
  "timestamp": "2026-05-20T10:00:03.000Z"
}
```

`bounce_type` is eusend's classifier bucket and is frequently `Uncategorized` — a real
rejection with no specific rule. It alone won't distinguish an invalid mailbox from a
reputation block.

- `smtp_code` — numeric reply code. Absent when the bounce arrived asynchronously as a DSN.
- `dsn_status` — dotted status (`5.1.1`), present only on async DSN bounces, in place of `smtp_code`.
- `diagnostic` — the remote server's verbatim reason, trimmed to 500 chars. Absent if none was sent.

**Branch on `bounce_type` or `smtp_code`. Never match on `diagnostic`** — wording varies by
provider and changes without notice. Treat it as text for humans and logs.

The address is suppressed automatically on a hard bounce; there is nothing to do to stop
sending to it.
