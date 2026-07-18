# Observability & Operations

The operational question for every flow: **"how would I know this broke?"** Anchors as of 2026-07-11.

## What's instrumented

| Signal | Mechanism | Location |
| --- | --- | --- |
| Error tracking | Sentry via `withReporting` (captures + rethrows, redacts args) | `sentryWrapper.ts:236`; wrapping `createBooking`, `getBookingData`, `handleWebhookTrigger` |
| Sentry config | per-runtime configs | `apps/web/sentry.*.config.ts`, `instrumentation*.ts` |
| Distributed tracing | trace context threaded through booking | `distributedTracing` (`RegularBookingService.ts:511-519`), `traceContext` on the request |
| Structured logging | `tslog` sub-loggers with prefixes | `logger`, `criticalLogger`; `slots/util.ts:78` `[slots/util]` prefix |
| Per-event verbose logging | opt-in by event type | `isEventTypeLoggingEnabled` (`RegularBookingService.ts:631-633`) raises log level |
| Task processing logs | `console.info` per attempt | `task-processor.ts:13-39` |
| PII-safe logging | redaction helpers | `getPiiFreeCalendarEvent`, `piiHasher` (`event.ts:39`), `safeStringify` |
| Uptime/synthetic checks | Checkly | `checkly.config.ts`, `__checks__/` |
| Health/cron | cron routes + workflows | `apps/web/app/api/cron/*`, `.github/workflows/cron-*.yml` |

## "How would I know it broke?" per flow

| Flow | Failure | Detection |
| --- | --- | --- |
| Booking create | throws → Sentry event with `functionName`, redacted args, trace id | Sentry alert; error rate |
| Availability | Redis timeout → logged, falls back to DB | log line (`slots/util.ts:117-120`); latency spike |
| Webhook delivery | subscriber 5xx → task retries, then stops | task `attempts`/`lastError` in DB; `task-processor` logs |
| Reservation | upsert fails → (misleadingly) "not found" | error rate — but message is wrong (fix in [ticket 2](../06-contribution-practice/01-good-first-tickets.md)) |
| Auth | session invalid → `UNAUTHORIZED` | 401 rate; Sentry |
| Cron jobs | job fails | workflow status; Checkly |

## Gaps & operations notes

- **No visible dead-letter/alert for exhausted tasks** — a webhook that fails `maxAttempts` times just stops; you'd only see it by querying `Task` rows with a high `attempts` and null `succeededAt`. Adding a dead-letter alert is a real improvement (see [03-architecture-and-patterns/06](../03-architecture-and-patterns/06-architecture-critique.md)).
- **Misleading error messages** (reserve handler) reduce diagnosability — accurate errors are an observability feature, not just UX.
- **PII discipline** is built in (`piiHasher`, `safeStringify`, redaction in `withReporting`) — when you add logging, route it through these; never log raw emails or `Credential.key`.

## Deploy / rollback

- **Migrations** deploy via `yarn db-deploy` (`prisma migrate deploy`); use expand/contract so a rollback of app code doesn't break on the new schema (see [03-architecture-and-patterns/02](../03-architecture-and-patterns/02-data-model-and-persistence.md)).
- **Feature flags** (`FeaturesRepository`, e.g. bot detection in `event.ts:28-30`) let you disable a behavior without a deploy — the safe rollback lever for risky changes.
- **Blast radius:** changes to `Booking`/`EventType` or the availability path affect the whole product; ship them small, behind flags, with the ability to revert.

## The mindset

Before merging a feature, answer three questions: (1) if this fails in production, what signal fires? (2) can I find the affected records? (3) can I turn it off without a deploy? If the answer to any is "no," add the missing observability or flag. That is the operational maturity interviewers listen for.

Drill: for a new "SMS reminder" feature, list the log line, the Sentry-captured error, the metric, and the flag you'd add so you'd know it broke and could disable it. Self-grade — **Strong:** all four, plus how you'd find failed reminders in the `Task` table.
