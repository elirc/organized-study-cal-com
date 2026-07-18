# Key Flows

Seven end-to-end flows, each with a trace table, the validation/authorization boundary, side effects, test coverage, and an interview angle. These flows are reused as examples across the whole curriculum — learn them once here. Line anchors verified 2026-07-11; trust the path + described shape if numbers drift.

Flows covered: (1) create a booking — server, (2) fetch available slots — read path, (3) reserve a slot — concurrency, (4) cancel a booking — authz, (5) public booking page render — SSR/UI, (6) webhook delivery — async side effect, (7) event-type update — auth boundary.

---

## Flow 1: Create a booking (the core write path)

Why this flow matters: it is the product. It touches validation, availability re-checking, a DB transaction, idempotency, calendar sync, and a notification fan-out — the whole backend in one request.

Open these files first:
- `apps/web/pages/api/book/event.ts` — the HTTP front door.
- `packages/features/bookings/lib/service/RegularBookingService.ts:486-760` — the orchestrator (`handler`).
- `packages/features/bookings/lib/handleNewBooking/createBooking.ts:60-146` — the DB write.
- `packages/prisma/extensions/booking-idempotency-key.ts` — how `idempotencyKey` is set.

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | Booker UI | `apps/web/modules/bookings/hooks/useBookings.ts:136-137` | React Query mutation, `mutationFn: createBooking` | form responses | client trust — never final |
| 2 | client lib | `packages/features/bookings/lib/create-booking.ts:5-14` | `POST /api/book/event` | `BookingCreateBody` | — |
| 3 | API route | `apps/web/pages/api/book/event.ts:20-40` | Turnstile (opt), bot detection, rate limit by hashed IP | `NextApiRequest` | rate-limit keyed on IP hash only |
| 4 | API route | `event.ts:42-58` | session (optional — anonymous allowed), resolve `RegularBookingService` from DI | `userId \| -1` | anonymous bookings are expected |
| 5 | service | `RegularBookingService.ts:523-549` | load event type, validate reschedule window, build zod schema from `eventType.bookingFields`, parse body | `bookingData` | schema is event-type-specific |
| 6 | service | `RegularBookingService.ts:580-629` | blocked-email check, spam check start, active-bookings-per-booker limit, email verification | — | throws `HttpError` early |
| 7 | service | `RegularBookingService.ts:659-699` | resolve reschedule/seat, requires-confirmation flags, dedupe existing booking (round-robin/unconfirmed) | `isConfirmedByDefault` | dedupe returns existing booking |
| 8 | service → availability | `handleNewBooking/ensureAvailableUsers.ts` | re-check the slot is actually free server-side | busy times | **the real double-booking guard** |
| 9 | write | `handleNewBooking/createBooking.ts:96-146` | `prisma.$transaction`: cancel-update original (if reschedule) + `booking.create` | `Booking` row | single transaction |
| 10 | Prisma extension | `booking-idempotency-key.ts:27-37` | on create with status ACCEPTED, set `idempotencyKey = uuidv5(start.end.userId)` | unique key | dedupe at DB layer |
| 11 | side effects | `RegularBookingService.ts` (EventManager, tasker, webhooks) | calendar create/update, emails/SMS, webhook triggers, no-show tasks | — | fan-out after commit |

Validation and authorization: input validation is schema-driven in `getBookingData` (`handleNewBooking/getBookingData.ts:96`) using fields declared on the event type; business guards are the throw-early checks at `RegularBookingService.ts:580-651`. There is **no owner authorization** — booking is a public action; the "authorization" is really *availability* + *rate limit* + *bot/spam* defenses.

Persistence and side effects: the only DB write is the `$transaction` at `createBooking.ts:138-145`. Idempotency is enforced by the unique `Booking.idempotencyKey` (`schema.prisma:855`) populated by the client extension — a retried identical request collides on the unique key instead of double-booking. Calendar/email/webhook effects happen **after** commit and can fail independently (see Flow 6 and [03-architecture-and-patterns/04-side-effects-async-and-reliability.md](../03-architecture-and-patterns/04-side-effects-async-and-reliability.md)).

Tests that cover it: unit tests in `packages/features/bookings/lib/handleNewBooking/test/` (`booking-limits.test.ts`, `booking-validations.test.ts`, `booking-flags.test.ts`) and `getBookingData.test.ts`; E2E in `apps/web/playwright/`. The 2,690-line service itself has no single test — coverage is at the seams.

What juniors usually miss: that the frontend only *showed* free slots — the server must re-check availability at step 8, because time passed and someone else may have booked. Trusting the client here is the classic bug.

What seniors notice: idempotency is implemented as a Prisma *client extension* (cross-cutting, invisible at call sites) rather than in the service — clever but a hidden control; the key excludes attendee identity, so it dedupes by (time, organizer) which is intentional for retries but means two *different* people cannot hold the exact same organizer slot as separate ACCEPTED rows (that's what seats/round-robin handle).

Interview angle: "How do you prevent double-booking / handle a form double-submit?" → idempotency key + server-side availability re-check + DB unique constraint. Concrete, three-layered, has a failure mode. See [08-interview-prep/03-api-and-data-modeling-questions.md](../08-interview-prep/03-api-and-data-modeling-questions.md) Q on idempotency.

Drill: list every place a booking could be created twice, and which layer stops it. Self-grade — **Basic:** names the unique key. **Solid:** adds the server re-check at step 8. **Strong:** explains what the key's inputs include/exclude and why retries are safe but concurrent *distinct* bookings need seats logic.

---

## Flow 2: Fetch available slots (the read path)

Why this flow matters: it's the most performance-sensitive read in the app, cached in Redis, and it composes ~10 subsystems through dependency injection.

Open these files first:
- `packages/trpc/server/routers/viewer/slots/getSchedule.handler.ts` (8 lines).
- `packages/features/di/containers/AvailableSlots.ts` (module wiring).
- `packages/trpc/server/routers/viewer/slots/util.ts:78-120` (cache wrapper).

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | tRPC | `getSchedule.handler.ts:5-8` | resolve `AvailableSlotsService` from container | input schema | — |
| 2 | DI container | `di/containers/AvailableSlots.ts:22-44` | ~15 modules loaded (repos + services) | tokens | duplicate `busyTimesModule` load, 36-37 |
| 3 | cache | `slots/util.ts:104-120` | `withSlotsCache`: key = `JSON.stringify(input)`, TTL 2000ms (line 79) | cached slots | key stability = property order |
| 4 | service | `slots/util.ts` `_getAvailableSlots` | qualified hosts → per-user availability → aggregate → subtract busy → apply booking limits → build slots | slot list | timezone/DST correctness |
| 5 | conflict | `conflictChecker/checkForConflicts.ts` | interval-overlap check against busy times | boolean | O(n) sorted scan |

Validation and authorization: input is zod-validated in `getSchedule.schema.ts`; this is a **public** read (no login needed to see a host's availability), so there's no authz — only correctness and caching concerns.

Persistence and side effects: read-only, plus a Redis get/set. A Redis timeout is caught and logged, not thrown (`util.ts:113-120`) — availability degrades to a DB computation rather than failing.

Tests that cover it: `slots/util.test.ts`, `slots/getBusyTimesFromLimitsForUsers.test.ts`, and timezone-dependent tests gated by `VITEST_MODE=timezone`.

What juniors usually miss: that the 2-second cache TTL means two users can fetch the *same* "available" slot within the window — availability is advisory, not a lock. The lock-ish behavior is Flow 3.

What seniors notice: the cache key is the stringified input; any nondeterministic property ordering or added field silently changes cache hit rate (see the [risk register](../09-reference/risk-register.md)).

Interview angle: "Design an availability service" — see [08-interview-prep/04-system-design-from-this-repo.md](../08-interview-prep/04-system-design-from-this-repo.md).

Drill: trace how a busy event on a connected Google Calendar becomes a removed slot. Self-grade — **Solid:** names `getBusyTimes` + `checkForConflicts`. **Strong:** explains cache TTL vs. correctness tradeoff.

---

## Flow 3: Reserve a slot (concurrency & the TOCTOU seam)

Why this flow matters: it's short, complete, and demonstrates a real time-of-check-to-time-of-use race — prime interview and review material.

Open: `packages/trpc/server/routers/viewer/slots/reserveSlot.handler.ts` (all 126 lines).

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | handler | `reserveSlot.handler.ts:26-29` | read/create `uid` cookie, compute `releaseAt = now + MINUTES_TO_BOOK` | uid string | — |
| 2 | handler | `:30-40` | load event type (seats?) | eventType | 404 if missing |
| 3 | handler | `:45-64` | seat logic: don't reserve unless it's a seated event with seats left | boolean | comment-heavy edge cases |
| 4 | **check** | `:67-75` | `findReservedByOthers` — is someone else holding this slot? | boolean | **read** half of TOCTOU |
| 5 | **use** | `:77-103` | if free, `upsert` a `SelectedSlots` row per user | rows | **write** half — no transaction |
| 6 | handler | `:111-122` | set `uid` cookie (SameSite None/Lax for embeds) | Set-Cookie | cross-origin embed logic |

Validation and authorization: zod input (`reserveSlot.schema.ts`); public action; the only "auth" is the per-visitor `uid` cookie identity.

Persistence and side effects: writes `SelectedSlots` rows (`schema.prisma:1437-1449`, unique on `[userId, slotUtcStartDate, slotUtcEndDate, uid]`). Rows expire via `releaseAt` and are cleaned up by a cron. **The check (step 4) and the write (step 5) are not in a transaction** — two concurrent reservations can both see "free" and both upsert (confirmed; see [09-reference/risk-register.md](../09-reference/risk-register.md)).

Tests that cover it: `reserveSlot.handler.test.ts`.

What juniors usually miss: that a reservation isn't a booking — it's a soft, expiring hold to reduce (not eliminate) collisions during checkout.

What seniors notice: three separate issues — the TOCTOU race, a `FIXME` at 80-82 (team events reserve against `eventType.users`, the wrong set), and a `catch` at 104-109 that rethrows a misleading "Event type not found" for any upsert failure. All three are review-kata gold.

Interview angle: "Walk me through a race condition you'd worry about." → this exact read-then-write. Fix menu: DB transaction + unique constraint, `SELECT ... FOR UPDATE`, or a single `upsert` that relies on the unique index. See [05-quality-engineering/03-systematic-debugging.md](../05-quality-engineering/03-systematic-debugging.md).

Drill: rewrite steps 4-5 to be race-safe using the existing unique constraint. Self-grade — **Strong:** your fix needs no read-before-write and explains why the unique index makes the upsert authoritative.

---

## Flow 4: Cancel a booking (authorization boundary)

Why this flow matters: cancellation is where *authorization* actually lives (unlike creation), with a layered guard chain — a clean example of "check state, then permission, then act."

Open: `packages/features/bookings/lib/handleCancelBooking.ts:80-179`.

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | handler | `handleCancelBooking.ts:93-105` | zod-parse input, load `bookingToDelete` | Booking | — |
| 2 | guard | `:122-127` | reject if already CANCELLED (avoid deleting a reschedule's calendar event) | — | idempotency-ish guard |
| 3 | guard | `:133-138` | reject if event type disallows cancelling | — | — |
| 4 | authz | `:140-146` | compute `isCancellationUserHost` (userId match or email match) | boolean | email-based host match |
| 5 | guard | `:148-160` | require reason if configured; reject if event already ended | — | — |
| 6 | authz | `:162-176` | seated events: require caller be host/owner unless seatReferenceUid | — | 401 on non-host |
| 7 | effects | `:178+` | fetch webhooks, `BOOKING_CANCELLED` trigger, calendar delete, refunds | — | fan-out |

Validation and authorization: this is the authz-heavy flow. Note the guards are ordered state-first (already cancelled, ended) then permission (host check). The host check mixes identity sources (`userId` vs `email === cancelledBy`) — a subtle boundary worth discussing.

Tests that cover it: search `handleCancelBooking` tests under `packages/features/bookings/lib/`.

What juniors usually miss: step 2 — you can't just delete; a CANCELLED booking may already have surrendered its calendar slot to a reschedule.

What seniors notice: authorization by email equality (`user.email === cancelledBy`, line 141) is weaker than by authenticated id; fine for a public cancel-link token flow, but you'd want to be sure `cancelledBy` is trustworthy.

Interview angle: "How do you structure authorization checks?" → state guards before permission guards; deny by default; single boundary. See [03-architecture-and-patterns/03-validation-auth-and-permissions.md](../03-architecture-and-patterns/03-validation-auth-and-permissions.md).

Drill: enumerate every way the guard chain can reject, with status code. Self-grade — **Solid:** lists 4+ guards. **Strong:** flags the email-vs-id authz mix and when it's safe.

---

## Flow 5: Public booking page render (SSR + UI state)

Why this flow matters: shows the App Router → legacy `getServerSideProps` bridge that pervades `apps/web`, plus zustand-based client state for the Booker.

Open:
- `apps/web/app/(booking-page-wrapper)/[user]/[type]/page.tsx` (73 lines).
- `apps/web/modules/bookings/components/Booker.tsx:55-80`.
- `packages/features/bookings/Booker/store.ts:442-708`.

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | RSC page | `[user]/[type]/page.tsx:18-33` | build legacy ctx from headers/cookies/params, run `getServerSideProps` via `withAppDirSsr` | props | async params (Next 15) |
| 2 | RSC page | `:36-70` | `generateMetadata` runs SSR again for SEO/robots | Metadata | duplicate data fetch |
| 3 | client | `Booker.tsx:55-100` | hydrate Booker with props: event, schedule, slots, form | props | large prop surface |
| 4 | client store | `Booker/store.ts:442-708` | zustand store holds selected date/time/layout/month, synced to URL query params | store | URL is source of truth |
| 5 | client | `hooks/useBookings.ts` | submit → Flow 1 | — | — |

Validation and authorization: per `AGENTS.md`, permission checks belong in `page.tsx`, never `layout.tsx`. Public pages have no auth; `isSEOIndexable`/`hidden` drive `robots` (`page.tsx:63-68`).

Tests that cover it: Playwright E2E under `apps/web/playwright/` drive the booking page.

What juniors usually miss: that `generateMetadata` and the page both call `getData`, i.e. SSR data fetch can run twice (Next dedupes via request memoization, but the pattern is worth noting).

What seniors notice: URL query params are the store's source of truth (`Booker/store.ts` query-param sync) — deep-linkable, back-button-safe state, but every state write must round-trip through the URL helpers.

Interview angle: "How do you manage complex client state?" → zustand store + URL as source of truth; when to use a store vs. React Query cache. See [08-interview-prep/02-frontend-framework-questions.md](../08-interview-prep/02-frontend-framework-questions.md).

Drill: find where selecting a time updates both the store and the URL. Self-grade — **Strong:** explains why URL-as-truth survives refresh and enables shareable links.

---

## Flow 6: Webhook delivery (async, signed side effect)

Why this flow matters: canonical outbound integration — signing, scheduling, retries — and the tasker abstraction.

Open:
- `packages/features/webhooks/lib/sendPayload.ts:296-327` (signing + send).
- `packages/features/tasker/tasker.ts` (queue interface).
- `packages/features/tasker/task-processor.ts:9-45` (retry loop).

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | booking service | `handleWebhookTrigger.ts:45` | on booking event, gather subscribers via `getWebhooks` | webhook list | — |
| 2 | schedule | `webhooks/lib/scheduleTrigger.ts` | immediate send or schedule (e.g. reminders, no-show) | task | — |
| 3 | tasker | `tasker.ts:35-41` | `create(type, payload, {scheduledAt, maxAttempts, referenceUid})` → `Task` row | Task | internal vs redis backend |
| 4 | processor | `task-processor.ts:11-40` | `getNextBatch`, run handler, `succeed` or `retry` with backoff | — | at-least-once delivery |
| 5 | send | `sendPayload.ts:301-327` | `fetch` POST with `X-Cal-Signature-256` HMAC, `redirect: "manual"` | Response | SSRF surface (subscriber URL) |

Validation and authorization: the payload is HMAC-SHA256 signed with the webhook secret (`sendPayload.ts:296-299`) so receivers can verify authenticity. `redirect: "manual"` (line 319) avoids following redirects — a small SSRF hardening.

Persistence and side effects: tasks are rows in the `Task` table (`schema.prisma:1825-1850`) with `attempts`/`maxAttempts`/`referenceUid` unique per type — enabling dedupe and retry. Delivery is **at-least-once**: a handler that half-succeeds then throws will be retried.

Tests that cover it: `sendPayload.test.ts`, `WebhookService.test.ts`, `handleWebhookScheduledTriggers.test.ts`.

What juniors usually miss: that the subscriber URL is attacker-influenced data — fetching it is an SSRF vector; `redirect: "manual"` and (ideally) URL allow-listing matter.

What seniors notice: at-least-once + retries means webhook receivers must be idempotent; the `referenceUid` unique constraint is how Cal.diy avoids scheduling duplicate tasks.

Interview angle: "How do you deliver webhooks reliably?" → queue + retries + signature + idempotent consumer + dead-letter after maxAttempts. See [03-architecture-and-patterns/04-side-effects-async-and-reliability.md](../03-architecture-and-patterns/04-side-effects-async-and-reliability.md).

Drill: what happens if the subscriber returns 500 on attempt 3 of 3? Trace `task-processor.ts:26-34`. Self-grade — **Strong:** describes the terminal state and how you'd add a dead-letter/alert.

---

## Flow 7: Update an event type (the authorization boundary you can break)

Why this flow matters: it's where the PBAC permission model lives — and where a **confirmed stub** currently short-circuits team-permission checks. Great for teaching "read the whole chain, don't trust a name."

Open:
- `packages/trpc/server/routers/viewer/eventTypes/_router.ts:104-109` (delete wiring).
- `packages/trpc/server/routers/viewer/eventTypes/util.ts:103-199` (`createEventPbacProcedure`).
- `packages/trpc/server/routers/viewer/eventTypes/util.ts:15-20` (the stub).

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | tRPC | `_router.ts:104-109` | `delete` uses `createEventPbacProcedure("eventType.delete", [ADMIN, OWNER])` | input | — |
| 2 | middleware | `util.ts:120-144` | load event (id, userId, teamId, users, team.members) | EventType | — |
| 3 | authz (personal) | `util.ts:151-158` | personal event: must be owner or assigned user | — | correct |
| 4 | authz (team) | `util.ts:159-167` | team event: `new PermissionCheckService().checkPermission(...)` | boolean | **stub returns `true`** |
| 5 | validate | `util.ts:178-196` | assigned users must be in team | — | correct |

Validation and authorization: personal-event authz is real (owner/assignee). **Team-event authz is not**: `PermissionCheckService.checkPermission` at `util.ts:15-20` is a local class whose methods return `true`/`[]` unconditionally. So step 4 always passes — a team member of any role could pass a check labeled ADMIN/OWNER-only (confirmed; blast radius limited by the assigned-users validation at step 5 and by whatever session gating exists upstream, which is **unverified**).

Tests that cover it: `eventTypes/__tests__/`.

What juniors usually miss: the procedure is *named* `PbacProcedure` and passes a permission string — it *looks* enforced. You must open the service to see it's a stub.

What seniors notice: this is exactly the kind of gap an EE→community fork introduces (the real PBAC lived in enterprise code). It's a first-class finding, a ticket, and an interview story all at once.

Interview angle: "Tell me about a security issue you found in a codebase." → this. See [08-interview-prep/06-behavioral-star-stories.md](../08-interview-prep/06-behavioral-star-stories.md) and the ticket in [06-contribution-practice/02-mid-level-feature-tickets.md](../06-contribution-practice/02-mid-level-feature-tickets.md).

Drill: write the guard you'd put at step 4 to restore role enforcement without PBAC. Self-grade — **Strong:** checks the caller's `team.members` role against `fallbackRoles` and denies by default.
