# Side Effects, Async & Reliability

A booking isn't done when the row is written — calendars must sync, emails/SMS send, webhooks fire, reminders schedule. This file maps those effects and the reliability machinery around them.

## Side-effect map (after a booking commits)

| Effect | Owner | Trigger point | Delivery |
| --- | --- | --- | --- |
| Calendar create/update | `EventManager.ts:287/615` | after DB commit | sync per credential, best-effort |
| Video meeting create | `EventManager.createVideoEvent:1079` | with calendar | best-effort |
| Confirmation email/SMS | `BookingEmailSmsHandler` + tasker | after commit | queued |
| Webhook (`BOOKING_CREATED`, ...) | `handleWebhookTrigger.ts:45`, `scheduleTrigger.ts` | after commit | queued, signed |
| No-show triggers | `scheduleNoShowTriggers.ts` | scheduled | tasker (delayed) |
| Reminders | cron + tasker | scheduled | tasker |
| Analytics/CRM | `tasker/tasks/analytics`, `crm` | after commit | queued |

The key architectural choice: **effects run after the DB commit, mostly via the tasker queue**, so a slow email never fails the booking.

## Idempotency (make retries safe)

- **Bookings**: `Booking.idempotencyKey @unique` (`schema.prisma:855`) set by a Prisma extension (`booking-idempotency-key.ts`) — a retried identical create collides instead of duplicating.
- **Tasks**: `Task @@unique([referenceUid, type])` (`schema.prisma:1846`) — scheduling the same task twice is a no-op when a `referenceUid` is provided.
- **Consumers must be idempotent**: because delivery is at-least-once (below), webhook receivers and task handlers must tolerate duplicates.

## Retries & the queue

The tasker (`packages/features/tasker/`) has two backends (`internal` DB-polling, `redis`) behind one typed interface (`tasker.ts:35-41`). The processor (`task-processor.ts:11-40`) pulls a batch, runs each handler, and on throw calls `Task.retry` with a backoff (`minRetryIntervalMins`), up to `maxAttempts` (`schema.prisma:1835`). `Promise.allSettled` (`:37`) isolates failures so one bad task doesn't abort the batch.

**Delivery guarantee: at-least-once.** A handler that half-succeeds then throws is retried. There's no visible dead-letter queue — after `maxAttempts` a task simply stops being retried (verify the exact terminal condition in `Task.retry`/`getNextBatch`), so poison payloads need monitoring.

## Webhooks: signing & transport hardening

`sendPayload.ts:296-327`: HMAC-SHA256 over the exact body (`X-Cal-Signature-256`), `redirect: "manual"` to avoid following redirects (SSRF hardening). The no-secret fallback signs `"no-secret-provided"` — receivers must reject that. The subscriber URL is attacker-influenced → an SSRF surface wanting allow-listing.

## Timeouts, backpressure, failure visibility

- **Redis timeout** on the slot cache is caught and logged, then falls back to DB computation (`slots/util.ts:113-120`) — graceful degradation.
- **Observability**: `withReporting` (`sentryWrapper.ts:236`) captures exceptions with `functionName` + redacted args and rethrows; the tasker logs each attempt (`task-processor.ts:13-39`).
- **Backpressure**: the DB-polling tasker processes fixed batches (`getNextBatch`), which naturally bounds concurrency but doesn't scale like a real broker.

## The reliability gap to discuss (senior signal)

Effects are enqueued **after** the booking transaction commits, in application code — not inside the transaction. So there's a window: booking committed, process dies before enqueue → the effect never happens. A **transactional outbox** (write the task row in the *same* transaction as the booking, a separate relay dispatches it) closes that gap. Cal.diy's `Task` table is most of what an outbox needs; the missing piece is same-transaction enqueue. This is the single best reliability improvement to propose in a design interview.

## Failure-mode checklist for any side effect

- [ ] Does it run after commit (so failure doesn't corrupt the DB state)?
- [ ] Is it retried, and is the consumer idempotent?
- [ ] Is there a terminal/dead-letter state and an alert?
- [ ] Is the external call timed out and its failure visible?
- [ ] For outbound HTTP: signed? URL allow-listed? redirects blocked?

Drill: trace a `BOOKING_CREATED` webhook from `handleWebhookTrigger.ts:45` to the signed `fetch` in `sendPayload.ts:312`, marking where a failure is retried vs. lost. Self-grade — **Strong:** you identify the post-commit enqueue gap and name the outbox fix.

Interview angle: "design reliable webhook delivery", "how do you run background jobs", "exactly-once vs at-least-once" → [08-interview-prep/03](../08-interview-prep/03-api-and-data-modeling-questions.md) and [system design](../08-interview-prep/04-system-design-from-this-repo.md) Step 5.
