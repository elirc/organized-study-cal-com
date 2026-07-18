# Refactor & Design Katas

Nine senior katas. Each has a target, a task, and self-grading criteria. Do them in your local tree; the goal is judgment, not a merged PR. Anchors as of 2026-07-11.

---

## Kata 1: Fix a boundary leak

Target: `createBooking.ts:129-136` calls `prisma.credential.findFirstOrThrow` directly inside service/write logic.

Task: move the credential existence check into a repository method (or the credential repository), keeping the service free of raw queries per `AGENTS.md`.

Self-grade — **Strong:** the service now depends on an injected repository method; the query moved without behavior change; you noted why (testability, layer purity) and added/kept a test.

---

## Kata 2: Introduce the transactional outbox (design)

Target: post-commit effect enqueue ([03/04](../03-architecture-and-patterns/04-side-effects-async-and-reliability.md)).

Task: write the design (no code) for writing the outbound `Task` inside the booking transaction plus a relay. Include the failure the current design has, the new invariant, and the fault-injection test.

Self-grade — **Strong:** you state the "committed-but-not-notified" gap precisely, keep at-least-once + idempotent consumers, and describe the kill-after-commit test.

---

## Kata 3: Split a module

Target: `RegularBookingService.ts` (2,690 lines).

Task: identify three cohesive extraction seams (e.g. reschedule validation, dedupe/existing-booking, side-effect orchestration). For one, write the characterization test that must stay green, then sketch the extraction.

Self-grade — **Strong:** extractions are behavior-preserving, each behind a green test, and you didn't accidentally drop a guard (e.g. the blocked-email check).

---

## Kata 4: Remove duplication

Target: the two authorization procedures `eventOwnerProcedure` (`util.ts:24-95`) and `createEventPbacProcedure` (`:103-199`) share load + assigned-users validation.

Task: extract the shared load-and-validate logic into one helper; keep the two procedures' different permission branches.

Self-grade — **Strong:** the duplication is gone, the personal-vs-team branches remain distinct, and tests cover both. You resisted merging behaviors that should stay separate.

---

## Kata 5: Improve type safety

Target: any spot using a loose type or a `!` non-null assertion (grep for `!` on array access, or `as` casts) in the booking path.

Task: replace one with a schema-validated type, an assertion function (like `assertNonEmptyArray`, `RegularBookingService.ts:123`), or `satisfies`.

Self-grade — **Strong:** the compiler now enforces the invariant the cast/`!` assumed; no `as any`; the change reads as documentation of the invariant.

---

## Kata 6: Design a migration

Target: add `EventType.maxBookingsPerDayPerBooker Int?` (a new limit).

Task: write the expand/contract plan (nullable-first add → read/write → enforce → optional backfill), the zod schema update, the repository `select` update, and the rollback.

Self-grade — **Strong:** nullable-first, no `NOT NULL` on a populated table, and you named the three code sites (schema, zod, repository) plus the enforcement point in the booking guards.

---

## Kata 7: Reduce an N+1

Target: find a query loop (grep for `await prisma.` inside `for`/`map`) — or design one away in the availability/host path.

Task: batch it into a single `select` with nested relations (like `createBooking.ts:103-118`).

Self-grade — **Strong:** query count drops from N+1 to 1-2; you measured (or reasoned) the win; the batched query uses `select`, not `include`.

---

## Kata 8: Write an RFC

Target: the waitlist feature (Project 4).

Task: write a one-page RFC (problem, proposal, alternatives, schema, risk, rollout, open questions) using the template in [07/02](../07-career-and-collaboration/02-writing-prs-and-rfcs.md).

Self-grade — **Strong:** a maintainer could approve or push back from your RFC alone; alternatives are real; risk + rollback are concrete.

---

## Kata 9: Review a flawed PR

Target: Katas 3, 4, and 8 in [04-code-reading-gym/04-review-katas.md](../04-code-reading-gym/04-review-katas.md).

Task: write the full review — findings categorized blocking/important/optional, each with a kind, specific comment and a suggested path.

Self-grade — **Strong:** you block only what must block, catch the security/reliability issue, and teach in the optional comments without nitpicking style as blocking.

---

## The senior meta-skill

Every kata rewards the same instinct: **change behavior only when you mean to, and prove it.** Characterization tests before refactors, failing-first tests before fixes, expand/contract before migrations, and small reversible PRs throughout. That instinct — more than any single technique — is what "senior" means, and what the design/refactor interview rounds test.
