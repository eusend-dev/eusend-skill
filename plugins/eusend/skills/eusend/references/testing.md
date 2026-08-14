# Testing

## Test-mode keys

A key starting `eu_test_` accepts sends, records them in the log, and fires the full webhook
lifecycle — but never delivers to a real inbox. Use one in development and CI.

By default a test send simulates `email.sent` → `email.delivered` → `email.opened` →
`email.clicked`, with opens and clicks following the message's tracking settings.

Webhook deliveries from a test send carry `test_mode: true` in the payload, so a shared
endpoint can tell them apart from production traffic.

## Simulating failures

The recipient's local part selects the outcome:

| Send to | Simulates |
| --- | --- |
| `bounced@…` | A hard bounce — fires `email.bounced` |
| `complained@…` | A spam complaint — fires `email.complained` |
| anything else | The normal delivered → opened → clicked path |

That covers the two branches worth testing: the ones that suppress an address and change
what your application should do next.

## What test mode does not do

- **Test keys cannot modify the suppression list.** They can read it. Sends still respect it.
- Domain verification still applies. Sending from an unverified domain fails with
  `DOMAIN_NOT_VERIFIED` on a test key just as it does on a live one.
- A `sending_access` restriction, including a `domain_id` pin, behaves the same in test mode
  as in production — so a scoped key behaves in CI exactly as it will in production.

## Testing a webhook handler

Point a test-mode webhook at a tunnel (`ngrok`, `cloudflared`) and send to `bounced@…` to
exercise the bounce path end to end.

To unit-test verification without any network, generate a signature the same way the server
does:

```ts
import { createHmac } from 'node:crypto'

function signWebhook(secret: string, id: string, timestamp: string, body: string) {
  return 'v1,' + createHmac('sha256', secret).update(`${id}.${timestamp}.${body}`).digest('base64')
}

const body = JSON.stringify({ type: 'email.bounced', email_id: 'test', tags: {}, timestamp: '…' })
const headers = {
  'webhook-id': 'msg_test_1',
  'webhook-timestamp': String(Math.floor(Date.now() / 1000)),
  'webhook-signature': signWebhook(secret, 'msg_test_1', String(Math.floor(Date.now() / 1000)), body),
}
```

Assert on both branches: a valid signature is accepted, and a tampered body is rejected with
401. The second is the one that catches a handler that parses before verifying — mutate the
body after signing and the test must fail.

## CI

Give CI its own `eu_test_` key rather than sharing a developer's. Test-mode sends do not
consume the monthly quota's live budget, but they do exercise rate limits — 100 requests per
10 seconds per organization — so a test suite that fans out hundreds of parallel sends will
hit `RATE_LIMITED`. Serialize, or back off.

## Local development without any key

The SMTP bridge and the API both need a real key. For fully offline work, keep email behind a
small interface and swap the implementation, rather than trying to fake the API — then use a
test key in CI to exercise the real thing.
