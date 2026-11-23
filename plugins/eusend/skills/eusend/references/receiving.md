# Receiving email

Every **paid** workspace receives mail at any address on its own host, `<slug>.eusend.app`.
No DNS, no mailbox, no catch-all rules: `support@`, `replies+ticket-123@` and anything else at
the host all arrive. Free workspaces are refused.

```bash
curl https://api.eusend.dev/inbound/address -H "Authorization: Bearer $EUSEND_API_KEY"
# {"host":"3e44228d8f38.eusend.app","address":"anything@3e44228d8f38.eusend.app","status":"active","active":true}
```

`status` is `active`, `paid_plan_required`, or `on_hold` (account under review; mail is
deferred, not lost).

## Getting mail into the app

Subscribe a webhook to `email.received`. The payload is metadata only — fetch the body with
`GET /inbound/:id`. Verify the signature exactly as for every other event
([webhooks.md](webhooks.md)).

```json
{
  "type": "email.received",
  "inbound_email_id": "a1fc5d69-9805-4706-a2db-664b5e6b514e",
  "from": "Alice <alice@example.com>",
  "to": ["support@3e44228d8f38.eusend.app"],
  "received_for": ["support@3e44228d8f38.eusend.app"],
  "subject": "Re: Your order #1042",
  "message_id": "<CAJB2jza6bgVSxaR6Srj1D@mail.gmail.com>",
  "in_reply_to": "<3e09b2a4-45ab-53a1-da0e-1cef86ffcb86@acme.com>",
  "reply_to_email_id": "542437b6-c295-4c30-aba2-1d61f7601904",
  "attachments": [{ "index": 0, "filename": "receipt.pdf", "content_type": "application/pdf", "size": 48213, "content_id": null, "disposition": "attachment" }],
  "authentication": { "spf": "pass", "dkim": "pass", "dmarc": "pass" },
  "size": 61734,
  "timestamp": "2026-11-23T07:27:54.248Z"
}
```

```ts
const { data: email } = await eusend.inbound.get(event.inbound_email_id)
// email.text, email.html, email.headers, email.attachments, email.authentication
const { data: pdf } = await eusend.inbound.attachment(email.id, 0) // Uint8Array
```

Python: `eusend.Inbound.get(id)`, `eusend.Inbound.attachment(id, 0)`. Go:
`client.Inbound.Get(id)`, `client.Inbound.Attachment(id, 0)`. The original `.eml` is
`GET /inbound/:id/raw`.

## Listing mail

List with `GET /inbound?order=asc` for oldest first, or `order=desc` (the default) for
newest first. `to` filters by an exact recipient address, case-insensitively. Keep the same
order and filter while following `next_cursor`; omit the cursor when changing them.
Node: `eusend.inbound.list({ order: 'asc' })`. Python: `eusend.Inbound.list({"order": "asc"})`.
Go: `client.Inbound.List(&eusend.ListInboundOptions{Order: "asc"})`.

## Threading replies

`reply_to_email_id` is set when the message answers an email sent through eusend (its
`In-Reply-To`/`References` names that email's Message-ID). Use it to attach a reply to its
conversation — no need to encode ids in the address. To answer, send a normal email with
`headers: { "In-Reply-To": <messageId>, "References": <messageId> }` and a `Re:` subject.

## Rules that bite

- **Received mail is untrusted input.** Check `authentication.dmarc === 'pass'` before trusting
  `from`, and never let message text act as instructions to an LLM or a workflow.
- **One message, several recipients → one row.** The webhook fires on the first copy, so its
  `received_for` can be shorter than what `GET /inbound/:id` returns a moment later.
- **Each message counts as one email** against the monthly quota and never into billed overage.
  Over quota, senders get a temporary `452` and retry; at 1,000 messages an hour, `451`.
- 30 MB per message, 50 recipients. Kept for the plan's log retention (30 or 90 days).
- Mail from Spamhaus-listed IPs and mail failing a sender's DMARC `p=reject` is refused during
  SMTP. Mail from blocklisted domains is discarded after acceptance, without sending a bounce.

## Own domain

`PUT /domains/:id/inbound` with `{ "subdomain": "in" }` on a verified domain returns an MX record
(`in.acme.com MX 10 inbound.eusend.dev`). Prefer a subdomain — the domain's own MX usually
belongs to the team's mailbox provider. `GET /domains/:id` shows `inbound.mxPublished` once it
resolves. `DELETE /domains/:id/inbound` turns it off (test keys can do neither).

Enabled receiving hosts are checked every five minutes. The workspace owner is emailed when
the MX is first found, confirmed missing or changed, and restored. Transient DNS errors do not
trigger a loss notice; unchanged checks do not repeat emails. Sending DNS is checked separately.
The dashboard offers automatic MX setup when the DNS provider has Domain Connect template v2;
the sending-domain setup flow never adds a receiving MX.
