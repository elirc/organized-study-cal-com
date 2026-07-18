# Interview Prep: API & Data Modeling Questions

Thirteen cards on API design, validation, auth, pagination, schema, transactions, and caching — most anchored to Cal.diy's API and schema. Concrete example + tradeoff + failure mode.

---

## Q1: How do you make a create endpoint idempotent?

Anchor: `Booking.idempotencyKey @unique` (`schema.prisma:855`) + Prisma extension (`booking-idempotency-key.ts`).
Junior: "check if it exists first." Mid: derive a deterministic key from the request, store under a unique index; a duplicate collides at the DB. Failure mode: app-level check-then-insert is racy — the constraint is the guarantee. Senior: key granularity matters — Cal.diy keys on (start, end, userId), so retries are safe but distinct attendees need seats logic; and it's enforced via an extension so no write path can forget it.
Follow-up: "what if the key is too coarse/fine?" Drill: contrast Booking's key with `Task @@unique([referenceUid, type])`.

---

## Q2: How do you prevent double-booking under concurrency?

Anchor: server re-check (`ensureAvailableUsers`), soft hold (`SelectedSlots`), unique key.
Junior: "unique constraint." Mid: three layers — advisory hold (UX), server availability re-check, DB constraint. Failure mode: the reservation is racy (`reserveSlot.handler.ts:67-103`) and the re-check can both pass under a true race. Senior: full serialization needs a transaction with a conflict predicate or `SELECT ... FOR UPDATE`; the current layers reduce but don't eliminate concurrent-distinct collisions — a deliberate tradeoff.
Drill: say which layer actually guarantees correctness.

---

## Q3: REST vs. RPC (tRPC) — when each?

Anchor: public booking = REST route; authed management = tRPC; third-party = NestJS REST (`apps/api/v2`).
Junior: "REST is standard." Mid: tRPC gives end-to-end types with no codegen for a JS client that deploys together; REST for public/anonymous/third-party/non-JS consumers. Failure mode: exposing tRPC to third parties (tight coupling, versioning pain). Senior: Cal.diy uses all three deliberately — internal tRPC, hardened public route, versioned platform REST.
Drill: why is `/api/book/event` REST, not tRPC?

---

## Q4: How do you validate API input?

Anchor: zod at the boundary (`getBookingData.ts:96`, `*.schema.ts`), aligned to Prisma via `@zod` annotations.
Junior: "if statements." Mid: schema-validate untrusted input once at the boundary, infer types with `z.infer`. Failure mode: validating after use, or `.passthrough()` letting junk through. Senior: the booking schema is dynamic (built from `bookingFields`), and `@zod` schema annotations keep DB and validation aligned so they can't drift.
Drill: how does a custom event-type question get validated?

---

## Q5: How do you design authorization for a multi-tenant API?

Anchor: `eventTypes/util.ts` load-then-check; the PBAC stub (`:15-20`).
Junior: "check the token." Mid: object-level checks — load the resource, verify ownership/role before acting (stops IDOR); scope by userId/teamId. Failure mode: "logged in" ≠ "allowed"; and a check named `PbacProcedure` that's actually a stub returning true. Senior: deny by default; the personal-event check here is correct, the team check is bypassed — a real finding.
Drill: describe the IDOR defense and the gap.

---

## Q6: How do you paginate a large list endpoint?

Anchor: `Booking` indexes (`schema.prisma:918-929`); `useBookingCursor`/`useBookingListData`.
Junior: "LIMIT/OFFSET." Mid: cursor (keyset) pagination for stable, performant paging on large tables; order by an indexed column. Failure mode: OFFSET on a big table is slow and skips/dupes under concurrent writes. Senior: cursor on `(userId, createdAt/id)` matches the indexes; handle ties deterministically.
Drill: which `Booking` index supports a per-user cursor?

---

## Q7: When JSON columns vs. normalized columns?

Anchor: `bookingFields`, `responses`, `metadata`, `bookingLimits` are `Json` (`schema.prisma:193,867,240,246`).
Junior: "JSON is flexible." Mid: JSON for variable, app-validated shapes (custom questions), columns for queryable/constrained data. Failure mode: querying/aggregating JSON is slow and untyped; drift. Senior: Cal.diy binds each JSON column to a zod schema via `@zod` to prevent drift — flexibility without losing validation, at the cost of queryability.
Drill: why is `responses` JSON but `status` a column?

---

## Q8: How do you change a schema without downtime?

Anchor: 596 migrations; `Booking`/`EventType` are hubs.
Junior: "add the column." Mid: expand/contract — add nullable/with default, backfill, dual-write, switch reads, drop old; never `NOT NULL` on a populated table in one step. Failure mode: a blocking migration or a rename-in-place breaking running code. Senior: name the code sites (schema, zod, repository `select`), add indexes concurrently, keep it reversible.
Drill: plan adding `Booking.internalReference String?`.

---

## Q9: When do you use a transaction, and what's its scope?

Anchor: `createBooking.ts:138-145` (`$transaction`: update original + create new).
Junior: "wrap DB calls." Mid: a transaction makes multi-statement writes atomic; keep it small and don't do slow/external work inside. Failure mode: an email/HTTP call inside a transaction ties commit to an external service. Senior: Cal.diy keeps the txn to the two DB writes and pushes effects to the tasker after commit — correct, though a transactional outbox would close the post-commit gap.
Drill: what's in the booking transaction vs. deliberately outside it?

---

## Q10: How do you cache a read path correctly?

Anchor: `withSlotsCache` Redis, 2s TTL, key `JSON.stringify(input)` (`slots/util.ts:79,104-120`).
Junior: "add Redis." Mid: cache with a TTL matched to freshness needs; on error, degrade to the source (this catches Redis timeouts and falls back). Failure mode: unstable cache key (property order) tanks hit rate; too-long TTL serves stale slots. Senior: canonicalize the key; consider event-driven invalidation on new bookings instead of a blunt TTL.
Drill: what's the risk of the current key, and the fix?

---

## Q11: How do you design reliable webhook delivery?

Anchor: HMAC signing (`sendPayload.ts:296-318`), tasker retries (`task-processor.ts`), `Task` dedupe.
Junior: "POST to their URL." Mid: sign (HMAC) so they can verify; queue with retries; dedupe scheduling with `referenceUid`; idempotent consumers. Failure mode: at-least-once → duplicates if the consumer isn't idempotent; SSRF via the subscriber URL. Senior: `redirect:"manual"` + allow-listing; dead-letter after maxAttempts; you can't make delivery exactly-once, so make consumption idempotent.
Drill: trace a duplicate delivery to its cause.

---

## Q12: How do you model recurring events / availability?

Anchor: `Schedule`/`Availability` (`@db.Time` + `timeZone`); slots computed, not stored.
Junior: "a table of times." Mid: store recurring rules (days + start/end) + a timezone, and *compute* concrete slots on demand minus busy times — storing every slot would be huge/stale. Failure mode: storing UTC without the zone → DST bugs. Senior: wall-clock + zone, convert at the edges, test transition days; `recurringEvent` JSON on the event type describes the recurrence.
Drill: why is there no "slot" table?

---

## Q13: How do you return good API errors?

Anchor: typed errors by layer (`HttpError`/`ErrorWithCode` in services, `TRPCError` in routers), one translation boundary.
Junior: "return 500." Mid: throw typed errors with status/codes at the point of failure; translate to transport at the edge; don't swallow. Failure mode: `catch {}` or a misleading message (`reserveSlot.handler.ts:104-109`) that misdirects debugging. Senior: rich errors + context via `withReporting` (redacts args), safe client messages, accurate codes (409 for conflict, 403 vs 401 correctly).
Drill: fix the misleading reserve error and say why.

---

## Drill routine

90 seconds each. The double-booking (Q2), authz (Q5), idempotency (Q1), and webhook (Q11) cards are the highest-value — they connect to [04-system-design](04-system-design-from-this-repo.md). Cross-links: [03-architecture-and-patterns](../03-architecture-and-patterns/README.md).
