# Audiences, broadcasts, and suppressions

## Audiences and contacts

```ts
const { data: audience } = await client.audiences.create('Newsletter')

await client.audiences.createContact(audience.id, {
  email: 'user@example.com',
  firstName: 'Jane',
  lastName: 'Smith',
})

// Bulk import — up to 1,000 per call
await client.audiences.batchCreateContacts(audience.id, {
  contacts: [
    { email: 'alice@example.com', firstName: 'Alice' },
    { email: 'bob@example.com', firstName: 'Bob' },
  ],
})

await client.audiences.listContacts(audience.id, { subscribed: true, search: 'gmail.com' })
await client.audiences.updateContact(audience.id, contactId, { unsubscribed: true })
await client.audiences.deleteContact(audience.id, contactId)
await client.audiences.list()
await client.audiences.delete(audience.id)
```

`createContact` upserts, so re-importing a list doesn't duplicate anyone.

## Broadcasts

A broadcast is one message to a whole audience. Creating and sending are deliberately
separate calls.

```tsx
const { data } = await client.broadcasts.create({
  name: 'May newsletter',
  audienceId: audience.id,
  from: 'hello@acme.com',
  subject: 'May update',
  html: '<p>Hi {{first_name}}…</p>',
  // or: react: <MayNewsletter />   or: templateId: '…'
})

await client.broadcasts.send(data.id) // now
await client.broadcasts.send(data.id, { scheduledAt: '2026-06-01T09:00:00.000Z' }) // later
await client.broadcasts.cancel(data.id) // if scheduled

await client.broadcasts.get(broadcastId) // includes delivery stats
await client.broadcasts.list()
await client.broadcasts.update(broadcastId, { subject: 'Updated subject' })
await client.broadcasts.delete(broadcastId)
```

Broadcasts have their own `scheduled_at` — the per-email scheduling on `POST /emails` does
not apply.

**Broadcasts never continue past the plan quota**, whatever the metered-billing setting says.
That only covers transactional `POST /emails` and `/emails/batch`.

### The unreviewed-account allowance

Until an account has been reviewed, broadcasts and very large batches reach **500
recipients** in total. Past that, the first 500 are delivered and the remainder is held:
the broadcast moves to `held` and returns `BROADCAST_HELD`.

- Retrying does not clear it, and sending again does not restart it — a cleared broadcast
  resumes from exactly where it stopped. (This is different from a _paused_ broadcast.)
- The allowance is per account and one-time. Once reviewed, it's lifted for good.
- Accounts sending from a domain registered in the last 30 days keep it until reviewed.
- Transactional sending via `POST /emails` is entirely unaffected.

The reason is shared outbound IP reputation: one purchased list degrades delivery for every
sender on the platform, and a 500-address sample shows real bounce and complaint rates
faster than a queue would. `support@eusend.dev` can clear it on a deadline.

## Unsubscribes

Broadcasts carry a footer link to a hosted page at
`https://api.eusend.dev/unsubscribe/:token`. Confirming removes the contact from that
audience. The contact record is kept; only the subscription status changes.

**Unsubscribed is not suppressed.** An unsubscribed contact is excluded from future
broadcasts to that audience, and nothing else. Transactional mail via `POST /emails` still
reaches them — which is usually correct for receipts and password resets, and wrong if the
user asked to hear nothing at all. For a total opt-out, add them to the suppression list
explicitly.

## Suppressions

Addresses the organization will not send to, across every API key and sending domain. Never
shared between organizations. A send to a suppressed address is skipped silently and recorded
with status `suppressed`; if _every_ recipient is suppressed, the request fails with
`ALL_SUPPRESSED` (422).

| Reason      | Added by                        |
| ----------- | ------------------------------- |
| `bounce`    | Automatically, on a hard bounce |
| `complaint` | Automatically, on a spam report |
| `manual`    | By you, via API or dashboard    |

```ts
await client.suppressions.list({ reason: 'bounce', limit: 50 })
await client.suppressions.list({ email: '@acme.com' }) // substring match
await client.suppressions.create({ email: 'opted-out@example.com' })
await client.suppressions.import([
  'one@example.com',
  { email: 'two@example.com', reason: 'complaint' },
])
await client.suppressions.remove('invalid@example.com') // by entry id or by address
const { data: csv } = await client.suppressions.export() // whole list, streamed CSV
```

Adding an already-suppressed address returns `200` with the **existing** entry — a manual add
never overwrites the reason, so it can't mask a real bounce. Import takes up to 1,000 per
call; items may be bare strings, so a CSV column works as-is. The response reports `count`,
`already_suppressed`, and `duplicates`.

Manual entries are capped at 100,000 per organization (`PLAN_LIMIT_EXCEEDED` past it).
Bounces and complaints are never refused.

Test-mode keys can read the list but not modify it.

### Two rules

**Import before the first send when migrating.** Export the previous provider's suppression
list and import it _before_ sending anything. Otherwise every address that already
hard-bounced there gets a fresh attempt from a new IP, which is the fastest way to wreck a
new sender's reputation. The dashboard's CSV importer handles SendGrid, Mailgun, Postmark,
and SES export formats.

**Removing an entry is a real decision.** Un-suppressing lets you mail an address that
bounced or complained. Do it when you know the mailbox was fixed or the complaint was a
mistake — never to retry a list that is failing. Doing it in bulk is what gets an account
suspended with `SENDING_SUSPENDED`.
