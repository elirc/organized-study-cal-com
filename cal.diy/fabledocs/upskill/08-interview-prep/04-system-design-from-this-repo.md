# System Design From This Repo: "Design a Scheduling Platform"

This is your flagship whiteboard exercise. You already have a reference implementation to cite — that's your edge. At each step, show what Cal.diy actually chose (with anchors) and what a simpler/stronger alternative looks like. Interviewers reward candidates who reason about tradeoffs with concrete evidence.

Prompt as an interviewer would give it: *"Design a Calendly/Cal.com-style scheduling app. Users publish event types; visitors book open time slots; the system syncs external calendars and notifies everyone."*

---

## Step 0 — Clarify requirements (always do this first)

Functional: publish event types with availability rules; show open slots for a public link; book/reschedule/cancel; sync busy times from Google/Outlook; send email/SMS confirmations and reminders; fire webhooks; optional payment. Non-functional: correctness (no double-booking), low-latency slot reads, timezone/DST correctness, reliability of notifications, multi-tenant isolation (teams).

Junior answer: jumps to tables. Mid-level: separates read (slots) from write (booking) and names double-booking + timezones as the hard parts. Senior: asks about scale (per-host QPS vs. global), consistency needs (is a soft-hold acceptable?), and which integrations are in scope.

Repo evidence this is the right framing: the two hardest subsystems in Cal.diy are exactly the slot **read** path (`slots/util.ts`, 1,435 lines) and the booking **write** path (`RegularBookingService.ts`, 2,690 lines).

---

## Step 1 — API sketch

Minimum surface:
- `GET /slots?eventType&start&end&timezone` → available slots (public).
- `POST /reserve` → soft hold (public).
- `POST /book` → create booking (public).
- `POST /cancel`, `POST /reschedule`.
- Authenticated CRUD for event types, availability schedules, webhooks.

What Cal.diy chose: a **hybrid**. Public writes go through Next.js API routes (`apps/web/pages/api/book/event.ts`) so they can layer bot/rate-limit/Turnstile defenses; authenticated management goes through **tRPC** (`packages/trpc/server/routers/viewer/*`) for end-to-end types; a separate **NestJS platform API** (`apps/api/v2`) exposes a versioned REST surface for third parties.

Alternative: single REST API (simpler, no tRPC coupling, but you lose type-safety and rebuild input validation). Or all-tRPC (great DX, awkward for public/third-party consumers and non-JS clients).

Mid-level adds: why booking is a plain API route, not tRPC — it must accept anonymous, IP-rate-limited, bot-checked traffic (`event.ts:20-40`). Senior adds: versioning strategy — Cal.diy versions the platform API (`apps/api/v2`) but not the internal tRPC/route surface, which is fine because internal callers deploy together.

---

## Step 2 — Data model

Core entities (from `schema.prisma`):
- `User` (401), `Team` (557) — tenants.
- `EventType` (156) — the bookable configuration: `length`, `minimumBookingNotice` (219), `periodType`/`periodDays` (future limits), `seatsPerTimeSlot`, `bookingLimits`/`durationLimits` (Json), `schedulingType`, `bookingFields` (custom questions).
- `Schedule` (945) + `Availability` (960) — recurring weekly rules (`days Int[]`, `startTime`/`endTime` as `@db.Time`) plus date overrides.
- `Booking` (851) — the fact: `uid`, `idempotencyKey @unique` (855), `status`, `startTime`/`endTime`, `responses` (Json), heavy indexing (918-929).
- `Attendee` (826), `BookingSeat` (1330), `Payment` (1095), `Webhook` (1142), `SelectedSlots` (1437, the soft holds), `Task` (1825, the job queue).

Design discussion points:
- **Availability is computed, not stored per-slot.** There is no "slot" table of open times — slots are derived at read time from `Schedule`/`Availability` minus busy times. This is the key modeling insight: storing every open slot would be huge and stale. Cite `slots/util.ts` building ranges via `buildDateRanges` and subtracting busy times.
- **Booking indexing** reflects the real queries: `[startTime, endTime, status]`, `[userId, status, startTime]`, `[eventTypeId, status]` (`schema.prisma:924-928`). Explain why: availability lookups and dashboards filter on exactly these.
- **JSON columns** (`bookingFields`, `responses`, `metadata`, `bookingLimits`) trade schema rigidity for flexibility; Cal.diy keeps them validated via zod (`@zod` annotations in the schema). Discuss the tradeoff: queryability vs. flexibility.

Junior: draws `User`, `EventType`, `Booking`. Mid-level: realizes slots are computed and adds the availability tables + indexes. Senior: discusses JSON-vs-columns, the timezone storage choice (`@db.Time` + per-schedule `timeZone`), and how teams/round-robin change the model.

---

## Step 3 — The availability read path (the interesting half)

Requirements: fast, correct across timezones/DST, reflects external calendar busy times, respects booking/duration limits and future/past bounds.

What Cal.diy chose (trace `slots/util.ts`):
1. Resolve qualified hosts (for team/round-robin).
2. For each host, compute availability from their schedule + date overrides + OOO.
3. Fetch **busy times** from connected calendars (Google/Outlook) and existing bookings.
4. Aggregate (for multi-host events), subtract busy, apply `bookingLimits`/`durationLimits` and period bounds (`isOutOfBounds`).
5. Slice into slots at `slotInterval`/`length`.
6. Cache the whole result in Redis for 2 seconds (`withSlotsCache`, `util.ts:104-120`, TTL at line 79).

Scaling & tradeoffs to raise:
- **Caching:** 2-second TTL absorbs bursts (a page load fires many slot queries) while keeping data fresh. Alternative: cache per-host-day with event-driven invalidation on new bookings — more complex, fresher.
- **External calendar latency** dominates; Cal.diy fetches busy times per request (with its own caching layers) — discuss a background sync that mirrors busy times locally (Cal.diy has calendar-subscription sync, `packages/features/calendar-subscription`).
- **Timezone correctness** is the classic bug source — storing times as `@db.Time` and applying the schedule's `timeZone` at compute time; tests are gated behind `VITEST_MODE=timezone` for a reason.

Mid-level adds the cache and the busy-time subtraction. Senior adds DST edge cases, the "cache TTL vs. correctness" dial, and moving external-calendar reads to a mirrored/subscribed model to cut p99 latency.

---

## Step 4 — The booking write path & double-booking

The hard invariant: **never confirm two conflicting bookings.** Three layers Cal.diy uses (walk them):
1. **Server-side availability re-check** at booking time (`ensureAvailableUsers`) — because the client's slot list is stale by the time they submit.
2. **Soft reservation** (`SelectedSlots`, Flow 3) — a TTL hold to reduce collisions during checkout (UX, not correctness; and note the TOCTOU race in `reserveSlot.handler.ts:67-103`).
3. **Idempotency key + unique constraint** (`Booking.idempotencyKey @unique`, populated by the Prisma extension) — the DB-level backstop against retries/double-submit.

Interviewer will push: "Two people book the same slot at the same millisecond — what happens?" Strong answer: the soft hold reduces the odds; the availability re-check may both pass (race); the *actual* guarantee comes from the unique constraint and/or a transaction — and Cal.diy's idempotency key is keyed on (start, end, organizer), so it dedupes identical retries but does **not** by itself serialize two *distinct* attendees — that's why seated events and round-robin exist, and why a stricter design would add a transactional conflict check or a `SELECT ... FOR UPDATE` on the host's bookings.

This is the single best tradeoff discussion in the whole exercise. Junior: "add a unique constraint." Mid-level: names all three layers. Senior: identifies that the current layers don't fully serialize concurrent distinct bookings and proposes the transaction/locking fix with its latency cost.

---

## Step 5 — Side effects & reliability

After commit, Cal.diy fans out: calendar create/update via `EventManager`, emails/SMS, webhooks, no-show triggers. Design points:
- **Do side effects after the DB commit**, asynchronously, via a queue (`Task` table + tasker, Pattern 9) so a slow email doesn't fail the booking.
- **At-least-once delivery** → consumers (and webhook receivers) must be idempotent; Cal.diy uses `Task @@unique([referenceUid, type])` to avoid duplicate scheduling.
- **Webhooks are signed** (HMAC, `sendPayload.ts:296-320`) and use `redirect: "manual"` (SSRF hardening).

Senior: raises the outbox pattern — right now effects are enqueued after commit in app code; a true transactional outbox would write the task in the *same* transaction as the booking to avoid "committed but never notified" gaps. Cite that as the improvement you'd make.

---

## Step 6 — Multi-tenancy, auth, and isolation

- Authn: NextAuth with Credentials/Google/Email providers (`next-auth-options.ts`), tRPC `isAuthed` middleware sets typed `ctx.user`.
- Authz: resource-scoped procedures load the record and check ownership/role (`eventTypes/util.ts`). **Flag the real gap:** the team PBAC check is a stub returning `true` (`util.ts:15-20`) — in a design interview, present this as "authorization must be enforced at the object level; a fork that removed the enterprise PBAC left a permissive stub, which is exactly the class of bug I'd add a test for."
- Tenant isolation: team events scope by `teamId` + membership; personal events by `userId`.

---

## Variation prompts (interviewer will pivot)

1. **"Now support teams with round-robin assignment."** → add `Host`/`Schedule` per member, a `getLuckyUser` selection with weights (`isRRWeightsEnabled`, `schema.prisma:260`), and qualified-host resolution in the slots path. Availability becomes the *union* of members' free times; assignment picks one at booking.
2. **"Add real-time 'this slot is being booked' indicators."** → the `SelectedSlots` holds already model this; push updates via SSE/websockets; discuss consistency of a soft signal.
3. **"Handle 10x traffic on popular booking links."** → the 2s Redis cache already helps reads; add per-host cache warming, mirror external busy times via subscriptions instead of live fetch, and rate-limit writes (already keyed on hashed IP, `event.ts:37-40`). Bottleneck is external calendar APIs and the booking transaction.
4. **"Guarantee no double-booking under concurrency."** → move the availability check + insert into one transaction with a conflict predicate, or add a unique/exclusion constraint on (host, time-range); accept the added latency and contention.
5. **"Support recurring bookings."** → `RecurringBookingService` + `recurringEventId` on Booking; discuss all-or-nothing vs. best-effort creation of the series.

---

## Scoring rubric (grade your mock)

- **Basic:** correct entities, one API, "add a unique constraint" for double-booking.
- **Solid:** separates read/write paths, computes slots (not stored), names caching + timezones, lists the three double-booking layers, puts side effects on a queue.
- **Strong:** identifies where the current design is *not* fully correct (reservation TOCTOU; idempotency key doesn't serialize distinct attendees; effects enqueued outside the booking transaction; PBAC stub), and proposes concrete fixes with their costs. Cites anchors from memory.

Practice drill: whiteboard steps 1-5 in 25 minutes out loud, then do variation prompt #4 in 5 minutes. Record yourself; if you can't name a repo anchor per step, re-read [01-codebase-cartography/05-key-flows.md](../01-codebase-cartography/05-key-flows.md).
