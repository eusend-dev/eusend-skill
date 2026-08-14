# Error codes

Every error response has the same shape:

```json
{ "error": "Invalid API key", "code": "UNAUTHORIZED" }
```

Reading the code: `error.name` (Node SDK), `e.code` (Python), `apiErr.Code` (Go).

## Full table

| Code | Status | Meaning | Action |
| --- | --- | --- | --- |
| `UNAUTHORIZED` | 401 | Key missing, invalid, or revoked | Fix the key. Never retry |
| `FORBIDDEN` | 403 | Key lacks permission for this action | Usually a `sending_access` key calling a read endpoint |
| `NOT_FOUND` | 404 | Resource does not exist | Never retry |
| `VALIDATION_ERROR` | 400 | Body failed validation | Read the message; fix the payload |
| `BAD_REQUEST` | 400 | Malformed request | Fix the payload |
| `CONFLICT` | 409 | Resource exists, or a scheduled email already started sending | Never retry |
| `RATE_LIMITED` | 429 | >100 requests / 10s per org | Back off ~seconds, retry |
| `DAILY_LIMIT_EXCEEDED` | 429 | Daily recipient ceiling hit | Retry after midnight UTC |
| `MONTHLY_LIMIT_EXCEEDED` | 429 | Monthly quota, or the 2× hard ceiling with metered billing on | Retry next month, upgrade, or enable metered billing |
| `PLAN_LIMIT_EXCEEDED` | 403 | Not allowed on this plan (domain limit, 100k manual suppressions) | Upgrade. Never retry |
| `DOMAIN_NOT_VERIFIED` | 403 | Sender domain not verified | Verify the domain. Never retry |
| `ALL_SUPPRESSED` | 422 | Every recipient is suppressed | Terminal for this send. Not an escalation |
| `SENDING_SUSPENDED` | 403 | Account suspended — high bounce/complaint rate over 7 days | Contact support@eusend.dev |
| `LIST_SEND_HELD` | 403 | Past the 500-recipient unreviewed list allowance | Wait for review. Retrying will not clear it |
| `BROADCAST_HELD` | 403 | Broadcast used its unreviewed allowance | Resumes from where it stopped once cleared. Do not re-send |
| `SERVICE_PAUSED` | 503 | Platform-wide pause | Retry shortly |
| `ATTACHMENT_STORAGE_ERROR` | 503 | Attachment could not be stored; the send did not happen | Retry |
| `PAYLOAD_TOO_LARGE` | 413 | Body over 16 MB | Shrink it. Base64 inflates files ~33% |
| `INTERNAL_ERROR` | 500 | Server error | Retry after a short delay |

## Retry policy

```ts
const RETRYABLE = new Set([
  'RATE_LIMITED',
  'SERVICE_PAUSED',
  'ATTACHMENT_STORAGE_ERROR',
  'INTERNAL_ERROR',
])

// Quota codes are retryable, but on a horizon of hours or days — never in a tight loop.
const DEFERRABLE = new Set(['DAILY_LIMIT_EXCEEDED', 'MONTHLY_LIMIT_EXCEEDED'])
```

Everything else is terminal: the request or the account needs changing, and retrying only
burns rate limit.

Always pair a retry with an `Idempotency-Key`. A retry after a timeout — where the send may
well have succeeded before the connection dropped — is exactly the case that double-sends
without one.

## The four codes people misread

**`ALL_SUPPRESSED` is not a failure of your code.** Every recipient is on the suppression
list because they hard-bounced, complained, or were added deliberately. Log it and move on.
Do not page anyone, and do not un-suppress to "fix" it.

**`LIST_SEND_HELD` and `BROADCAST_HELD` are not rate limits.** Recipients within the
500-address allowance were delivered; the rest waits on a one-time manual review. Retrying
does nothing. A held broadcast resumes from where it stopped when cleared — sending it
again does not restart it, and re-sending a broadcast that was merely paused is a different
operation. Email support@eusend.dev if it's urgent.

**`MONTHLY_LIMIT_EXCEEDED` has two meanings.** With metered billing off (the default), it's
the plan quota and sending has stopped. With it on, it's the hard ceiling at twice the
quota, which only support can raise. The message says which.

**`FORBIDDEN` on a read is usually the key, not the plan.** A `sending_access` key may only
call `POST /emails`, `POST /emails/batch`, `PATCH /emails/:id`, and `POST /emails/:id/cancel`.
Listing emails or domains with one returns 403.

## Language specifics

**Python** — exception subclasses of `eusend.EusendError`: `MissingApiKeyError`,
`InvalidApiKeyError`, `ValidationError`, `NotFoundError`, `RateLimitError`,
`ApplicationError`. All three 429 codes raise `RateLimitError`, so branch on `e.code` to
tell a rate limit from an exhausted quota. Network failures that never reach the server
raise `ApplicationError` with `status_code is None`.

**Go** — every failure, including network errors, is an `*eusend.Error`. Code constants are
exported (`eusend.CodeMonthlyLimitExceeded`, …). `StatusCode` is `0` for a network failure.
On a 429, `RetryAfter`, `RateLimitReset`, and `RateLimitRemaining` are parsed from the
response headers for you.

**Node** — nothing throws on an API error; the failure is in the returned `error`. Network
failures do throw, so wrap in `try/catch` *and* check `error`.
