# Sending

`POST /emails` returns an ID immediately; delivery is asynchronous. Never block a
user-facing action on it.

## Full option set

| Field (API / Node) | Notes |
| --- | --- |
| `from` | Required. Bare address or `"Acme <hello@acme.com>"`. Must be a verified domain |
| `to` | Required. String or array, max 50 |
| `cc` / `bcc` | Max 50 each |
| `reply_to` / `replyTo` | String or array, max 50 |
| `subject` | Required unless a template supplies one |
| `html` | HTML body |
| `text` | Plain-text body — always include one alongside `html` |
| `react` | Node only. A React Email element, rendered to HTML locally before sending |
| `template_id` / `templateId` | Saved template. Supplies the HTML — an `html` passed alongside is **ignored**. A `text` you pass is still used |
| `variables` | Template substitutions |
| `headers` | Custom headers |
| `tags` | Max 10. Charset `A–Z a–z 0–9 _ -` only |
| `track_opens` / `trackOpens` | Omit to use the org default (on unless changed) |
| `track_clicks` / `trackClicks` | Omit to use the org default |
| `attachments` | Max 20, 10 MB combined |
| `scheduled_at` / `scheduledAt` | ISO 8601, `Date`, or natural language. Max 30 days out |

At least one of `html`, `text`, `react`, or `template_id` is required.

The API is snake_case. The Node SDK is camelCase; Python takes snake_case dicts; Go uses
Go-cased struct fields (`ReplyTo`, `TemplateId`, `ScheduledAt`).

## The three SDKs, same send

```ts
// Node
const { data, error } = await client.emails.send({
  from: 'Acme <hello@acme.com>',
  to: 'customer@example.com',
  subject: 'Your order is confirmed',
  html: '<p>Thanks!</p>',
  text: 'Thanks!',
})
if (error) return handle(error.name, error.message)
return data.id
```

```python
# Python
import eusend
from eusend import EusendError

try:
    email = eusend.Emails.send({
        'from': 'Acme <hello@acme.com>',
        'to': 'customer@example.com',
        'subject': 'Your order is confirmed',
        'html': '<p>Thanks!</p>',
        'text': 'Thanks!',
    })
    return email['id']
except EusendError as e:
    handle(e.code, e.message)
```

```go
// Go
sent, err := client.Emails.SendWithContext(ctx, &eusend.SendEmailRequest{
	From:    "Acme <hello@acme.com>",
	To:      []string{"customer@example.com"},
	Subject: "Your order is confirmed",
	Html:    "<p>Thanks!</p>",
	Text:    "Thanks!",
})
if err != nil {
	var apiErr *eusend.Error
	if errors.As(err, &apiErr) {
		handle(apiErr.Code, apiErr.Message)
	}
	return
}
return sent.Id
```

Every Go method has a `WithContext` variant; the plain form uses `context.Background()`.
Optional pointer fields have helpers: `eusend.Bool(true)`, `eusend.String("x")`.

## Idempotency

```ts
await client.emails.send(payload, { idempotencyKey: `receipt-${orderId}` })
```

```bash
-H "Idempotency-Key: receipt-1234"
```

Any unique string up to 255 chars. A duplicate returns `200` with the original email ID and
queues nothing; the first send returns `201`. Derive the key from a domain object, not a
random UUID generated at call time — a UUID minted per attempt makes every retry a new send,
which defeats the point.

## Attachments

Each item needs `filename` plus exactly one of:

- `content` — base64-encoded bytes (Node also accepts `Uint8Array`)
- `path` — a public URL fetched at send time

Optional: `content_type` (inferred from the filename or fetched response), and `content_id`
to embed inline and reference from HTML as `<img src="cid:logo">`.

```ts
attachments: [
  { filename: 'invoice.pdf', content: base64Pdf, contentType: 'application/pdf' },
  { filename: 'logo.png', path: 'https://cdn.acme.com/logo.png', contentId: 'logo' },
]
```

Limits: 20 per message, 10 MB combined, inside a 16 MB request body. Base64 inflates files
by ~33%, so a 12 MB file will not fit. `ATTACHMENT_STORAGE_ERROR` (503) means the send did
**not** happen — retry it.

Attachments are not supported on the batch endpoint.

## Scheduling

```ts
const { data } = await client.emails.send({
  ...payload,
  scheduledAt: 'tomorrow at 9am', // or an ISO string, or a Date
})

await client.emails.update(data.id, { scheduledAt: '2026-07-04T09:00:00Z' }) // reschedule
await client.emails.cancel(data.id) // cancel; quota is refunded
```

Natural language is parsed server-side, and relative phrasings resolve in **UTC** — `"9am"`
is not the user's local morning. Pass an explicit ISO timestamp when the exact moment
matters.

Max 30 days out. All checks — domain verification, suppression, daily and monthly limits —
run at schedule time, and the send is debited from the monthly quota immediately (cancelling
refunds it). Recipients suppressed between scheduling and send time are still dropped.

Once sending has started, reschedule and cancel both return `409 CONFLICT`. Scheduling is
per-email; the batch endpoint doesn't support it, and campaigns use broadcasts instead.

## Batch

```ts
const { data } = await client.batch.send([ /* up to 100 messages */ ])

data.data.forEach((result, i) => {
  if ('error' in result) {
    logger.warn({ code: result.code, index: i }, 'batch item rejected')
  } else {
    record(result.id)
  }
})
```

One result per input, in the same order — `data[i]` describes `emails[i]`. A queued item has
`id`; a rejected one has `error` and `code`. **Failed items never fail the batch**, so a
`201` status tells you nothing about individual messages. Retry only the failures.

Same schema as a single send, except no `attachments` and no `scheduled_at`. Every message
counts toward daily and monthly quotas — a batch of 50 uses 50.

## Templates

```ts
const { data } = await client.templates.create({
  name: 'Welcome email',
  subject: 'Welcome, {{name}}!',
  html: '<h1>Hi {{name}}</h1><p>Welcome to {{product}}.</p>',
})

await client.emails.send({
  from: 'hello@acme.com',
  to: 'user@example.com',
  templateId: data.id,
  variables: { name: 'Jane', product: 'Acme' },
})
```

Placeholders are `{{variable}}`. The template supplies the HTML and the subject; a `subject`
passed on the send overrides it, an `html` passed alongside is ignored, and a `text` part you
pass is still used.

Also: `.list()`, `.get(id)`, `.update(id, {…})`, `.delete(id)`.

## React Email (Node only)

```tsx
await client.emails.send({
  from: 'hello@acme.com',
  to: 'user@example.com',
  subject: 'Welcome',
  react: <WelcomeEmail name="Jane" />,
})
```

Rendered to HTML locally before the request — the JSX never travels over the wire. Requires
`react` and `@react-email/render` as peer dependencies. Works on `templates.create` and
`broadcasts.create` too.

## Reading the log

```ts
const { data } = await client.emails.get(id)   // status + full event history
const { data: list } = await client.emails.list({ status: 'delivered', limit: 50 })
list.nextCursor // pass as `cursor` for the next page
```

Filter by `tag`: `GET /emails?tag=category:password_reset`, or `?tag=category` for any value.
Repeat the parameter to require several. Combines with `status`, `from`, `to`, and cursor
pagination.

Statuses: `queued`, `scheduled`, `canceled`, `sending`, `sent`, `delivered`, `bounced`,
`complained`, `suppressed`, `failed`. Note `suppressed` — the send was accepted but the
recipient was on the suppression list, so nothing was sent.

Both read endpoints require a `full_access` key.

## SMTP bridge

For an app that already speaks SMTP, change four settings rather than rewriting call sites.
Same pipeline: DKIM, tracking, suppression, and limits apply identically.

| Setting | Value |
| --- | --- |
| Host | `smtp.eusend.dev` |
| Port | `465` — implicit TLS. **Not** STARTTLS; 587 and 25 are not supported |
| Username | the literal string `eusend` |
| Password | any API key, `eu_live_…` or `eu_test_…` |

**The username is always `eusend`, not an email address and not the key.** Putting the key in
the username field is the most common cause of a `535` authentication failure.

Tags over SMTP use an `X-Eusend-Tag` header, which may repeat or carry comma-separated pairs:

```
X-Eusend-Tag: category=password_reset
X-Eusend-Tag: tier=pro, region=eu
```

Pairs that break the charset rules are dropped and the message still sends — SMTP has no
good way to return a validation error, and a bounced invoice is worse than an untagged one.
