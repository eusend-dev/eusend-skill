# Domains and DNS

A sending domain must be verified before `from` may use it. Unverified senders fail with
`DOMAIN_NOT_VERIFIED` (403).

## Add a domain

```ts
const { data } = await client.domains.create('acme.com')
data.records // every record to publish, in order
data.dkim    // { type: 'TXT', name: 'eusend._domainkey.acme.com', value: 'v=DKIM1; …' }
data.dmarc   // { type: 'TXT', name: '_dmarc.acme.com', value: 'v=DMARC1; p=none; …' }
```

```bash
curl -X POST https://api.eusend.dev/domains \
  -H "Authorization: Bearer $EUSEND_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "name": "acme.com" }'
```

eusend generates a 2048-bit RSA DKIM keypair per domain. Publish the records, then trigger
verification:

```ts
await client.domains.verify(domainId)
```

`POST /domains/:id/verify` moves the domain to `verified` or `failed`. DNS typically
propagates in under an hour but can take up to 48.

Rest of the surface: `client.domains.list()`, `.get(id)`, `.delete(id)`.

## The records

### DKIM — required

| Type | Host | Value |
| --- | --- | --- |
| `TXT` | `eusend._domainkey.acme.com` | `v=DKIM1; k=rsa; p=<public key>` |

This is what authenticates your mail. Nothing sends without it.

### SPF — do NOT add one to the root domain

**This is the most common mistake, and it causes real damage.**

SPF authenticates the envelope sender (the bounce address), not the `From:` header.
eusend's mail carries its own bounce domain, so an SPF record on `acme.com` is never
consulted for it — adding `include:_spf.eusend.dev` there authorises nothing.

Worse, a domain may publish only **one** SPF record. A second one makes SPF fail for
*every* service sending from that domain, including the company's own Google Workspace or
Microsoft 365 mail. If an SPF record already exists, leave it exactly as it is.

DMARC passes on DKIM alignment alone, so nothing is missing.

### DMARC — recommended

| Type | Host | Value |
| --- | --- | --- |
| `TXT` | `_dmarc.acme.com` | `v=DMARC1; p=none; rua=mailto:dmarc@acme.com` |

Start at `p=none` — it only collects reports and changes nothing about delivery. Move to
`p=quarantine`, then `p=reject`, once aggregate reports confirm legitimate mail is
authenticating. Every service sending on the domain's behalf must authenticate before
`p=reject`, or their mail starts bouncing.

`rua` must be an address someone actually controls; receiving servers send daily XML
digests there.

### Return-Path records — optional, both or neither

| Type | Host | Value | Priority |
| --- | --- | --- | --- |
| `TXT` | `send.acme.com` | `v=spf1 include:_spf.eusend.dev ~all` | — |
| `MX` | `send.acme.com` | `feedback.eusend.dev` | `10` |

Publishing **both** switches outbound mail to a bounce address on the customer's own
domain. SPF then passes *and* aligns, so DMARC passes on two mechanisms instead of one,
and delayed bounces return per-domain so failed addresses are suppressed automatically.

This SPF record is safe precisely because it sits on the `send.` subdomain, which has no
mail of its own and cannot collide with the root domain's SPF.

Detection is automatic — publish them and alignment switches on at the next verification.
No dashboard action.

## The sandbox sender

No DNS access yet? `onboarding@sandbox.eusend.dev` sends without any domain setup — but it
delivers **only to the account owner's own email address**, and is capped per day. It
exists to try the API. It is not a staging environment, and a `sending_access` key pinned
to a `domain_id` cannot use it.

## DMARCbis

DMARC became a full IETF standard in May 2026 (RFC 9989–9991, replacing RFC 7489). The
record eusend generates uses only long-stable tags and is DMARCbis-compatible; nothing
already published breaks. Some tags were deprecated — check
<https://eusend.dev/docs/domains/dmarc> before hand-writing a record with unusual tags.
