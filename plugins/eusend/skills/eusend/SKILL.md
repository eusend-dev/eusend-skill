---
name: eusend
description: Integrate and operate eusend, the EU-native transactional email API. Use when sending email from application code, wiring an eusend SDK (@eusend_dev/sdk, the eusend PyPI package, or eusend-go), setting up a sending domain and its DKIM/SPF/DMARC records, verifying eusend webhook signatures, handling bounces and suppressions, relaying through the eusend SMTP bridge, or debugging an eusend error code such as DOMAIN_NOT_VERIFIED, ALL_SUPPRESSED, LIST_SEND_HELD, or MONTHLY_LIMIT_EXCEEDED. Also use when adding transactional email (password resets, receipts, notifications) to a project that already depends on eusend.
license: MIT
---

# eusend

EU-native transactional email API. Base URL `https://api.eusend.dev`. Every request
authenticates with a bearer API key; mail is rendered, DKIM-signed and delivered from
European infrastructure.

## Pick the right interface

| Situation                                                                                                           | Use                                                                   |
| ------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------- |
| Application code sending mail                                                                                       | An official SDK — Node.js, Python, or Go                              |
| A runtime with no SDK                                                                                               | The HTTP API directly (JSON, bearer token)                            |
| A framework already configured for SMTP (Django, Rails, Laravel, WordPress, Ghost, Supabase, Nodemailer, PHPMailer) | The SMTP bridge — change host/port/credentials, don't rewrite the app |
| An assistant operating the account interactively                                                                    | The eusend MCP server (this plugin bundles it)                        |

Reach for the SMTP bridge when a working mailer config already exists. Swapping four
settings beats rewriting every call site, and it relays through `POST /emails` anyway,
so tracking, suppression, and limits behave identically.

## Setup

```bash
npm install @eusend_dev/sdk     # Node.js 18+ / Bun
pip install eusend              # Python 3.8+
go get github.com/eusend-dev/eusend-go
```

Read the key from the environment — `EUSEND_API_KEY` is the convention all three SDKs
honour when constructed with no argument. Never hardcode a key or commit one.

```ts
import { Eusend } from '@eusend_dev/sdk'
const client = new Eusend() // reads EUSEND_API_KEY
```

Keys are `eu_live_…` or `eu_test_…`. Use a test key for development and CI: it accepts
sends, records them, and fires the full webhook lifecycle, but never delivers to a real
inbox. See [references/testing.md](references/testing.md) for simulating bounces and
complaints.

**Give an application server a `sending_access` key, not `full_access`.** A
`sending_access` key can only send, reschedule, and cancel. `full_access` can read
`GET /emails/:id`, which returns the rendered body and the full recipient list — the
half of a leaked key actually worth stealing. A send-only key can also be pinned to one
domain with `domain_id`.

## A correct send

```ts
const { data, error } = await client.emails.send({
  from: 'Acme <hello@yourdomain.com>',
  to: 'customer@example.com',
  subject: 'Your order is confirmed',
  html: '<p>Thanks for your order!</p>',
  text: 'Thanks for your order!',
  tags: { category: 'order_confirmation' },
})

if (error) {
  // error.name is the machine-readable code, e.g. 'DOMAIN_NOT_VERIFIED'
  logger.error({ code: error.name, status: error.statusCode }, error.message)
  return
}
return data.id
```

The three SDKs report failure differently — match the host language, don't paper over it:

| SDK     | Failure shape                                                                                     |
| ------- | ------------------------------------------------------------------------------------------------- |
| Node.js | Returns `{ data, error }`. **Never throws on an API error.** `error.name` holds the code.         |
| Python  | Raises a subclass of `eusend.EusendError`. Branch on `e.code`.                                    |
| Go      | Returns `(result, error)`. Use `errors.As(err, &apiErr)` for `*eusend.Error`, then `apiErr.Code`. |

The Node SDK not throwing is the one people get wrong: a bare `await client.emails.send(...)`
with no `error` check silently drops every failed send.

Always send a `text` part alongside `html`. A message with no plain-text alternative is a
measurable spam signal.

## Rules that bite

These are the mistakes that actually reach production. Check each one.

**`from` must be on a verified domain.** Anything else fails with `DOMAIN_NOT_VERIFIED`
(403). No retry will fix it. To send before DNS is ready, use
`onboarding@sandbox.eusend.dev` — a shared sandbox that delivers **only to the account
owner's own email address**, capped per day. It is for trying the API, not for staging.

**Do not add an SPF record to the root domain.** eusend authenticates with DKIM, and its
mail carries its own bounce domain, so `include:_spf.eusend.dev` on `acme.com` authorises
nothing. It also breaks things: a domain may publish only **one** SPF record, so adding a
second makes SPF fail for every other service sending from that domain — including the
company's own Google Workspace or Microsoft 365 mail. If an SPF record already exists,
leave it alone. Optional `send.` subdomain records give SPF alignment safely; see
[references/domains-dns.md](references/domains-dns.md).

**Over SMTP, the username is the literal string `eusend`.** Not an email address, not the
API key — the key goes in the password field. Host `smtp.eusend.dev`, port `465`, implicit
TLS (not STARTTLS; 587 and 25 are not supported). Key-in-username is the usual cause of a
`535` failure.

**Retry with an `Idempotency-Key`, or not at all.** The send endpoint is not idempotent
on its own. Derive the key from your domain object so a retry is genuinely the same send:

```ts
await client.emails.send(payload, { idempotencyKey: `order-confirmation-${orderId}` })
```

A duplicate key returns `200` with the original email ID and queues nothing. The first
send returns `201`.

**Verify webhook signatures against the raw body.** Parsing JSON first and
re-serializing it produces a different byte string and the signature will never match.
The scheme is HMAC-SHA256 with the secret used as-is (not Svix's `whsec_` format); compare in constant time, and check length
before `timingSafeEqual` or a truncated header crashes the handler instead of returning
false. Full working handler in [references/webhooks.md](references/webhooks.md).

**Tags are labels, not data.** Allowed characters are `A–Z a–z 0–9 _ -` only — no dots,
spaces, `@`, or non-ASCII. An email address, a timestamp, or a UUID will be **rejected**.
Max 10 tags per message. Tag the _kind_ of mail (`category: password_reset`) and keep
identifiers in your own system; a high-cardinality tag makes a filter that matches one
email, which the email ID already does better.

**Batch sends fail per item.** `POST /emails/batch` returns `200`/`201` overall with one
result per input, in order. Each is either `{ id }` or `{ error, code }` — a rejected item
does not fail the batch. Iterate the results; don't assume success from the status code.
Batch does **not** support `attachments` or `scheduled_at`; send those individually. It
does honour `Idempotency-Key`, applied per item by position, so a retry must keep the same
items in the same order.

**Unsubscribed ≠ suppressed.** Unsubscribing removes a contact from broadcasts to that
audience. It does **not** stop transactional mail from `POST /emails`. If a user opts out
of everything, add them to the suppression list explicitly.

**Migrating from another provider? Import their suppression list before the first send.**
Otherwise addresses that already hard-bounced elsewhere get a fresh attempt from a new IP,
which is the fastest way to damage a new sender's reputation.

**Don't branch on bounce `diagnostic` text.** It is the remote server's verbatim wording
and changes without notice. Branch on `bounce_type` or `smtp_code`; treat `diagnostic` as
opaque text for humans and logs.

**Sending has ceilings beyond the API rate limit.** 100 requests / 10 seconds per
organization, plus a daily recipient ramp on new accounts (2,000 in the first 24 hours,
rising over ~7 days) and a monthly plan quota. A brand-new account cannot send at full
volume on day one — build the backoff in rather than discovering it at launch.

**List sending is reviewed.** Until an account is reviewed, broadcasts and very large
batches reach 500 recipients; the remainder is held (`LIST_SEND_HELD` / `BROADCAST_HELD`).
Retrying does not clear it. Transactional `POST /emails` is unaffected.

## Errors: retry or don't

`error.name` (Node), `e.code` (Python), `apiErr.Code` (Go). Full table in
[references/error-codes.md](references/error-codes.md).

**Retry with exponential backoff:**

| Code                       | Status | Note                             |
| -------------------------- | ------ | -------------------------------- |
| `RATE_LIMITED`             | 429    | Short backoff; the window is 10s |
| `SERVICE_PAUSED`           | 503    | Platform-wide pause, transient   |
| `ATTACHMENT_STORAGE_ERROR` | 503    | The send did not happen          |
| `INTERNAL_ERROR`           | 500    |                                  |

**Retry later, on a much longer horizon** — a tight loop just burns quota:
`DAILY_LIMIT_EXCEEDED` (resets midnight UTC), `MONTHLY_LIMIT_EXCEEDED` (resets monthly).

**Never retry — fix the request or the account instead:**
`VALIDATION_ERROR`, `BAD_REQUEST`, `PAYLOAD_TOO_LARGE`, `UNAUTHORIZED`, `FORBIDDEN`,
`NOT_FOUND`, `CONFLICT`, `DOMAIN_NOT_VERIFIED`, `ALL_SUPPRESSED`, `PLAN_LIMIT_EXCEEDED`,
`SENDING_SUSPENDED`, `LIST_SEND_HELD`, `BROADCAST_HELD`.

`ALL_SUPPRESSED` (422) means every recipient is on the suppression list. That is the
system working — treat it as a terminal outcome for that send, not an error to escalate.

Never block a user-facing action on a send completing. `POST /emails` returns an ID and
delivers asynchronously; a checkout that awaits delivery confirmation will hang.

## Limits worth knowing before you design around them

|                    |                                    |
| ------------------ | ---------------------------------- |
| Recipients         | 50 each for `to`, `cc`, `bcc`      |
| Batch              | 100 messages per request           |
| Attachments        | 20 per message, 10 MB combined     |
| Request body       | 16 MB (base64 inflates files ~33%) |
| Scheduling         | 30 days out, max                   |
| Tags               | 10 per message                     |
| Contact import     | 1,000 per call                     |
| Suppression import | 1,000 per call                     |

## References

Load these as needed — don't read them all up front.

- [references/sending.md](references/sending.md) — full send options, attachments,
  scheduling, templates, batch, React Email, and Python/Go equivalents of everything above.
- [references/domains-dns.md](references/domains-dns.md) — adding a domain, the DNS
  records to publish, verification, and what SPF/DKIM/DMARC each do.
- [references/webhooks.md](references/webhooks.md) — event types, payload shapes, a
  complete verified handler in three languages, and secret rotation.
- [references/audiences-broadcasts.md](references/audiences-broadcasts.md) — audiences,
  contacts, broadcasts, unsubscribe handling, and the suppression list.
- [references/error-codes.md](references/error-codes.md) — every code, its status, and
  what to do about it.
- [references/testing.md](references/testing.md) — test-mode keys, simulating bounces and
  complaints, and testing webhook handlers.

Official docs: <https://eusend.dev/docs>.

## Operating an account with the MCP server

This plugin bundles the eusend MCP server, which exposes the API as tools — sending,
listing emails, adding domains, managing audiences and broadcasts, reading suppressions.
It needs `EUSEND_API_KEY` in the environment. Set `EUSEND_FROM` to pin a default sender
so a send never has to guess an address.

Destructive operations are deliberately absent: no deleting domains, audiences, contacts,
or keys, and no un-suppressing. Broadcasts are two steps — `create_broadcast` only drafts,
`send_broadcast` is a separate call — so a bulk send is never one accidental tool call.
Use the API or the dashboard for anything the tools don't cover, and prefer a test-mode
key when experimenting.
