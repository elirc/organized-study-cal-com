# Behavioral STAR Stories

Ten STAR worksheets sourced from the work in [06-contribution-practice](../06-contribution-practice/README.md) and from studying this repo. Fill each in fully, rehearse under 2 minutes, end with impact. Even if you complete the ticket only locally, "I studied/contributed to an open-source scheduling platform" is a legitimate, strong story.

STAR = Situation, Task, Action, Result. The senior signal is in the Action (tradeoff weighed, risk reduced) and a quantified Result.

---

## Story 1: Found and fixed an authorization bypass

Prompts: "a mistake you caught", "a security issue", "attention to detail".
Source: the PBAC stub ([risk register #1](../09-reference/risk-register.md)).
Situation: reviewing team event-type permissions in a Cal.com fork. Task: verify ADMIN/OWNER-only deletes were enforced. Action: traced the named `PbacProcedure` to its implementation and found a stub returning `true`; wrote a failing-first test proving a non-admin could delete; implemented a role check against `team.members` with deny-by-default behind a flag. Result: closed a team-wide authorization gap; role-matrix tests prevent regression.
Evidence: `eventTypes/util.ts:15-20`; the failing test.
Senior-signal: read the implementation not the name; failing-first; flag-gated rollout.
Resume bullet: "Identified and remediated a team-authorization bypass in an OSS scheduling platform, adding role-matrix test coverage."
Rehearsal check: under 2 min? ends with impact? yes.

---

## Story 2: Fixed a race condition

Prompts: "a hard bug", "concurrency", "technical tradeoff".
Source: reservation TOCTOU (Ticket 2).
S: two users could both hold the same slot. T: make holds authoritative. A: identified the check-then-upsert race, chose to rely on the `SelectedSlots` unique index (removing the read) over a heavier transaction, documented the hold as advisory, added a concurrency integration test. R: eliminated the race with minimal blast radius.
Evidence: `reserveSlot.handler.ts:67-103`.
Senior-signal: chose the lighter fix and justified it.
Bullet: "Eliminated a slot-reservation race using an authoritative unique-index upsert."

---

## Story 3: Improved system reliability

Prompts: "went above the ask", "reliability", "ownership".
Source: transactional outbox (Project 2 / Ticket 3).
S: booking notifications could be lost if the process died after commit. T: guarantee delivery. A: proposed and prototyped writing the outbound task inside the booking transaction with a relay, keeping at-least-once + idempotent consumers; fault-injection test (kill after commit). R: closed a committed-but-not-notified gap.
Evidence: `createBooking.ts:138-145`; the outbox design.
Senior-signal: reliability reasoning beyond "add retries".
Bullet: "Designed a transactional outbox to make booking notifications crash-safe."

---

## Story 4: Learned a large, unfamiliar codebase fast

Prompts: "learned something complex", "onboarding", "ambiguity".
Source: mapping the booking + availability flows.
S: a 2,700-line booking service and a 1,400-line availability engine, no single test. T: understand it well enough to contribute. A: traced the write and read paths file-by-file, built trace tables, wrote docs for the next person, verified every claim against code. R: contributed fixes within days and produced onboarding docs.
Evidence: [key flows](../01-codebase-cartography/05-key-flows.md).
Senior-signal: systematic, verification-first, left it better documented.
Bullet: "Onboarded to a complex OSS codebase and authored the flow documentation the team lacked."

---

## Story 5: Disagreed with a technical decision

Prompts: "conflict", "disagreement", "influence".
Source: the reservation fix approach (unique index vs. transaction).
S: two valid fixes for the race. T: pick one. A: argued for the unique-index approach with evidence (removes the read entirely), heard the seat-edge-case concern, and committed to the transaction approach when the maintainer preferred it — both fix the race. R: shipped a fix with team buy-in.
Senior-signal: evidence-based, then disagree-and-commit.
Bullet: "Drove a concurrency-fix decision through evidence and consensus."

---

## Story 6: Made something testable

Prompts: "improved quality", "refactor", "tradeoff".
Source: dependency injection (Ticket 13) / service decomposition (Project 5).
S: a function imported a singleton, blocking tests. T: make it testable without breaking callers. A: inverted the dependency with a default arg, added a fake in tests, kept behavior identical (characterization test first). R: enabled unit testing of a previously-untestable path.
Senior-signal: characterization test before refactor; backward-compatible.
Bullet: "Improved testability of a booking helper via dependency injection, with no behavior change."

---

## Story 7: A performance win

Prompts: "optimization", "measurement".
Source: cache-key canonicalization (Ticket 4) / busy-time mirroring (Project 3).
S: availability recomputed often. T: raise cache hit rate. A: measured hit rate, found the `JSON.stringify` key varied with property order, canonicalized it, re-measured. R: higher hit rate, lower DB load — measured, not guessed.
Evidence: `slots/util.ts:108-109`.
Senior-signal: measured before and after.
Bullet: "Raised availability cache hit-rate by canonicalizing a non-deterministic cache key."

---

## Story 8: Improved observability after an incident

Prompts: "a failure you learned from", "operations".
Source: dead-letter alerting (Ticket 8) / reserve error message (Ticket 2).
S: a task could exhaust retries silently; a handler reported a misleading error. T: make failures visible. A: added a terminal state + alert for exhausted tasks; fixed the misleading error to preserve the real cause. R: failures now discoverable and accurately reported.
Senior-signal: treats diagnosability as a feature.
Bullet: "Added dead-letter visibility and accurate error reporting to a background-job pipeline."

---

## Story 9: Shipped a feature end to end

Prompts: "a project you owned", "full-stack".
Source: waitlist (Project 4).
S: no recourse for fully-booked slots. T: design a waitlist. A: schema (nullable-first migration), cancellation hook, async promotion via the tasker (idempotent, no double-promotion), Booker UI, opt-in flag. R: captured demand and auto-booked on cancellation.
Senior-signal: full-stack + idempotency + flagged rollout.
Bullet: "Designed and built a booking waitlist across schema, async promotion, and UI."

---

## Story 10: Balanced speed vs. correctness

Prompts: "a tradeoff", "pragmatism".
Source: the 2-second availability cache.
S: availability is expensive to compute. T: keep it fast without badly misleading users. A: reasoned about the TTL — 2s absorbs page-load bursts while keeping a booked slot from showing "free" for long; paired it with a graceful conflict UX at submit (Ticket 6). R: fast reads with acceptable, handled staleness.
Evidence: `slots/util.ts:79`.
Senior-signal: names the tradeoff and mitigates the downside.
Bullet: "Balanced availability latency and correctness with a short cache TTL plus graceful conflict handling."

---

## How to prepare

Pick 6 that cover the common prompt families — **conflict** (5), **ambiguity/learning** (4), **mistake** (1), **technical tradeoff** (2, 3, 10), **ownership/leadership** (9). Write each out, rehearse to under 2 minutes, and make sure every one ends on a concrete Result. Practice with [07-two-week-cram-plan.md](07-two-week-cram-plan.md).
