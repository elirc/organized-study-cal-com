# Architecture Critique

"What I'd change owning this for 3 months." Confirmed issues are separated from hypotheses. This file doubles as system-design interview material — cross-linked from [08-interview-prep/04](../08-interview-prep/04-system-design-from-this-repo.md).

## Strongest design choices (defend these in an interview)

1. **Computed availability, not stored slots.** No slot table; slots are derived from schedules minus busy times (`slots/util.ts`). Avoids a huge, always-stale table. This is the right call and the core insight of the domain.
2. **Thin transport, fat service, isolated repositories** with DI. `event.ts` → `RegularBookingService` → `BookingRepository`, wired via containers (`di/containers/AvailableSlots.ts`). Testable, reusable across web/tRPC/platform API.
3. **Idempotency as a DB invariant** via a Prisma extension (`booking-idempotency-key.ts`) — the guarantee holds no matter which code path writes a booking.
4. **Typed background queue** (`tasker`) with two interchangeable backends and typed payloads (`tasker.ts:6-27`). Side effects are off the request path and retried.
5. **End-to-end types** from Prisma → zod (`@zod` annotations keep DB and validation aligned) → tRPC inference. Casts are banned; the compiler carries contracts.
6. **Signed, redirect-hardened webhooks** (`sendPayload.ts:296-320`).

## Risks & tradeoffs (prioritized)

### Confirmed issues
1. **Team PBAC is a stub** (`eventTypes/util.ts:15-20`) — team-role authorization is bypassed. *Highest priority.* Migration path: implement a real permission service reading `team.members` roles; add failing-first tests; roll out behind a flag defaulting to deny. Blast radius: all team event-type mutations. Test strategy: per-role matrix tests.
2. **Reservation TOCTOU** (`reserveSlot.handler.ts:67-103`) — check-then-write race. Fix: single authoritative upsert on the unique index, or a transaction; clarify soft-hold semantics in a comment. Low blast radius (holds are advisory), quick win.
3. **Effects enqueued after commit, not in-transaction** — the reliability gap (see [04](04-side-effects-async-and-reliability.md)). Fix: transactional outbox using the existing `Task` table. Medium effort, high reliability payoff.
4. **Misleading reserve error** (`reserveSlot.handler.ts:104-109`) and **magic-link maxAge mismatch** (`next-auth-options.ts` ~362) — small correctness/security cleanups.

### Hypotheses (investigate before acting)
5. **Slot cache key = `JSON.stringify(input)`** (`slots/util.ts:104-120`) — key stability depends on property order; may hurt hit rate. Investigate actual key variance; canonicalize if needed.
6. **Webhook SSRF** — subscriber URL is fetched server-side (`redirect:"manual"` helps). Investigate whether private-range/DNS-rebind protection exists; add allow-listing if not.
7. **Giant service file** — `RegularBookingService.ts` is 2,690 lines with no single test (coverage at the seams). Not a bug, but a maintainability/blast-radius concern; the decomposition into `handleNewBooking/*` helpers is the right direction to continue.

## The 3-month plan (what I'd actually do)

- **Weeks 1-2:** land the PBAC fix (#1) with a role-matrix test suite — it's the one with real security impact. Ship the two small cleanups (#4).
- **Weeks 3-4:** make the reservation write race-safe (#2) and add concurrency-focused integration tests.
- **Weeks 5-8:** transactional outbox (#3) — write task rows inside the booking transaction, add a relay, prove no "committed-but-not-notified" gap with a fault-injection test.
- **Weeks 9-12:** investigate #5-#6 (cache key, SSRF), and continue decomposing the booking service (#7) with characterization tests before each extraction.

Guardrails: every change small and reversible (`AGENTS.md` PR limits), behind flags where behavior changes, with a failing-first test.

## Migration/test strategy per major recommendation

| Recommendation | Migration path | Test strategy | Rollback |
| --- | --- | --- | --- |
| Real PBAC | new permission service, flag-gated deny-by-default | per-role matrix; failing-first | flip flag |
| Race-safe reserve | swap to authoritative upsert | concurrency integration test | revert handler |
| Transactional outbox | enqueue in booking txn + relay | fault-injection (kill after commit) | keep old post-commit path behind flag |
| Cache key | canonicalize serialization | key-stability unit test | revert to stringify |

## Interview framing

When asked "critique this architecture," lead with the strengths (you understand *why* it's built this way), then give **prioritized** risks with fixes, costs, and tests — not a laundry list. The #1 PBAC finding shows you read implementations, not names; the outbox point shows you understand reliability beyond "add retries." That combination is a senior signal in a mid-level loop.

Drill: pick one confirmed issue and write its full ticket (problem, blast radius, plan, test, rollback) using the template in [06-contribution-practice/02](../06-contribution-practice/02-mid-level-feature-tickets.md).
