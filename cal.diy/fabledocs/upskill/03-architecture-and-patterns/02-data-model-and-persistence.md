# Data Model & Persistence

The schema (`packages/prisma/schema.prisma`, 2,851 lines) is the backbone. There are **596 migrations** in `packages/prisma/migrations/` — this is a long-lived, heavily-evolved schema, and reading a few recent migrations (e.g. `20260319000000_drop_workflow_tables`) shows how EE features were removed for the fork. Anchors as of 2026-07-11.

## Key entities & relationships

| Entity | Line | Role | Notable |
| --- | --- | --- | --- |
| `User` | 401 | account / host | auth, timezone, credentials |
| `Team` | 557 | tenant grouping | members, roles |
| `EventType` | 156 | bookable config | `length`, `bookingFields` (Json), `seatsPerTimeSlot`, `schedulingType`, limits |
| `Booking` | 851 | scheduled instance | `uid`, `idempotencyKey @unique`, `status`, heavy indexes |
| `Attendee` | 826 | person on a booking | email, timezone, locale |
| `Schedule` | 945 | named availability set | `timeZone` |
| `Availability` | 960 | weekly rule | `days Int[]`, `startTime/endTime @db.Time` |
| `SelectedSlots` | 1437 | soft reservation | `releaseAt`, unique `[userId, start, end, uid]` |
| `Credential` | 308 | integration secret | `key Json` — **never expose** (`AGENTS.md`) |
| `Webhook` | 1142 | callback subscription | `subscriberUrl`, `secret`, triggers |
| `Payment` | 1095 | payment record | linked to booking |
| `Task` | 1825 | background job | `attempts`, `maxAttempts`, unique `[referenceUid, type]` |

Relationships: `User` 1—* `EventType` 1—* `Booking` 1—* `Attendee`; `EventType` *—1 `Schedule` 1—* `Availability`; `Booking` *—* effects (`Payment`, `BookingReference`, `WebhookScheduledTriggers`).

## The modeling insight: slots are computed, not stored

There is **no "open slot" table**. Slots are derived at read time from `Schedule` + `Availability` minus busy times (`slots/util.ts`). Storing every open slot would be enormous and instantly stale. This is the single most important data-modeling decision in the product and a great interview point.

## Indexes reflect real queries

`Booking` carries targeted composite indexes (`schema.prisma:918-929`): `[startTime, endTime, status]`, `[userId, status, startTime]`, `[eventTypeId, status]`, `[userId, endTime]`, `[userId, createdAt]`. Each maps to a real access pattern — availability lookups, dashboards, per-host lists. The lesson: **index for the queries you actually run**, and a composite index serves prefixes too (so `[userId, status, startTime]` also helps `WHERE userId=`).

## JSON columns: flexibility vs. queryability

`bookingFields`, `responses`, `metadata`, `bookingLimits`, `locations`, `recurringEvent` are `Json`. Tradeoff: flexible, schema-migration-free evolution, but not efficiently queryable and easy to let drift. Cal.diy mitigates the drift with `@zod` annotations on the JSON columns (`schema.prisma:867`, `193`, `240`) so a validation schema is bound to each. Discuss this tradeoff in interviews: when to use JSON (variable, app-validated shape) vs. columns (queryable, constrained).

## Transactions & consistency

The only transaction on the booking write path is `createBooking.ts:138-145`: it optionally updates the original (reschedule cancel) and creates the new booking **atomically**. Side effects (calendar, email, webhook) happen *after* commit and are **not** in the transaction — so a booking can commit and a downstream effect still fail (mitigated by the tasker's retries). A stricter design would use a transactional outbox (write the task in the same transaction).

Consistency expectations: the booking is the source of truth; availability and reservations are eventually/soft consistent (2s cache, expiring holds).

## Safely changing the schema

The zero-downtime playbook (transferable):
1. **Add nullable / with default** — never a `NOT NULL` column without a default on a populated table.
2. **Migrate in steps**: add column → backfill → start writing → switch reads → drop old. Expand/contract, never rename-in-place on a live table.
3. **Generate the migration** with `yarn db-migrate` (prisma `migrate dev`), review the SQL, and apply with `yarn db-deploy`.
4. **Indexes**: add concurrently in production-scale DBs; justify each with a real query (indexes cost writes/storage).
5. **Respect the extensions**: a new booking-write column interacts with `booking-idempotency-key.ts` — check whether it belongs in the key.

Blast radius: `Booking` and `EventType` are referenced everywhere; a change there touches services, repositories, zod schemas, and the API. Small, reversible steps.

Drill: propose the migration to add a `Booking.internalReference String?` — write the steps, say why nullable-first, and name one place that must be updated (the repository `select`, the zod schema, or the create path). Self-grade — **Strong:** expand/contract plan + backfill + rollback.

Interview angle: "How do you migrate a schema without downtime?" and "When JSON vs columns?" → [08-interview-prep/03](../08-interview-prep/03-api-and-data-modeling-questions.md).
