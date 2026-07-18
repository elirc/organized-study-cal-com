# Mid-Level Feature Tickets

Twelve cross-layer tickets (schema/API/UI/tests). Each requires **design notes before implementation** and includes risk + rollback. These are the tickets that produce your best interview stories. Anchors as of 2026-07-11.

Per `AGENTS.md`: split into multiple PRs by layer where >500 lines / >10 files. Draft PRs, conventional commits, type-check + tests before pushing.

---

## Ticket 1: Implement real team-role authorization (fix the PBAC stub)

Difficulty: Hard — 3-5 days. Layers: tRPC middleware + tests. Skills: authz, security.

Story: As a team owner, I want only ADMIN/OWNER members to update/delete team event types.

Design notes (write first): `eventTypes/util.ts:15-20` is a stub returning `true`. Without the enterprise PBAC package, implement a minimal role check reading `team.members` role against the procedure's `fallbackRoles`. Decide: fail closed (deny if role missing). Enumerate every procedure using `createEventPbacProcedure` (`_router.ts`) — they all inherit the fix.

Acceptance criteria:
- [ ] Team-event mutations enforce `fallbackRoles`; non-privileged members get `FORBIDDEN`.
- [ ] Failing-first test (Recipe 4, [05/02](../05-quality-engineering/02-writing-tests-here.md)) passes after the fix.
- [ ] Per-role matrix test (OWNER/ADMIN/MEMBER × update/delete).

Read first: `eventTypes/util.ts:103-199`, `_router.ts:104-226`. Risk: could over-restrict legitimate actions — mitigate with the matrix test and a flag defaulting to enforce. Rollback: feature flag to revert to permissive (temporarily) if it blocks users. Blast radius: all team event-type mutations.

Interview story potential: "I found and fixed an authorization bypass left by an enterprise-code removal, with a role-matrix test suite." (Top-tier security STAR.)

---

## Ticket 2: Make slot reservation race-safe

Difficulty: Medium-Hard — 2-3 days. Layers: tRPC handler + repository + integration test.

Story: As a booker, I want a slot hold to be authoritative so two people don't both think they hold it.

Design notes: `reserveSlot.handler.ts:67-103` does check-then-upsert (TOCTOU). Options: (a) drop the read, rely on the `SelectedSlots` unique index via a conditional upsert; (b) wrap check+write in a transaction. Decide the intended semantics (soft advisory vs. exclusive) and document it. Also address the `FIXME` (`:80-82`, wrong users for team events).

Acceptance criteria:
- [ ] No read-before-write race; behavior documented.
- [ ] Concurrency integration test (two parallel reserves) asserts the intended outcome.

Read first: `reserveSlot.handler.ts`, `SelectedSlots` model (`schema.prisma:1437`). Risk: changing hold semantics affects the "someone's booking this" UX. Rollback: revert handler. Blast radius: reservation only (advisory), low.

Interview story potential: "I fixed a TOCTOU race in slot reservation and clarified the hold semantics." (Concurrency STAR.)

---

## Ticket 3: Transactional outbox for booking side effects

Difficulty: Hard — 1 week. Layers: service + persistence + tasker.

Story: As an operator, I want every committed booking to reliably trigger its notifications, even if the process dies right after commit.

Design notes: today effects are enqueued after the booking transaction ([03/04](../03-architecture-and-patterns/04-side-effects-async-and-reliability.md)). Write the task row **inside** the booking transaction (`createBooking.ts:138-145`), and let the existing tasker relay dispatch it. The `Task` table is most of an outbox already.

Acceptance criteria:
- [ ] Booking + its outbound task row commit atomically.
- [ ] Fault-injection test: kill after commit, verify the task still dispatches.

Risk: schema/interface change to the write path (high blast radius) — do it behind a flag, keep the old post-commit path until proven. Rollback: flag. Interview story potential: "I closed a committed-but-not-notified reliability gap with a transactional outbox." (Reliability STAR — senior signal.)

---

## Ticket 4: Canonicalize the slots cache key

Difficulty: Medium — 2 days. Layers: availability service + test.

Story: As a user, I want consistent availability performance regardless of request property order.

Design notes: `slots/util.ts:108-109` keys on `JSON.stringify(args.input)`. Replace with an explicit, sorted-key canonical string. Measure hit-rate before/after to justify.

Acceptance criteria:
- [ ] Deterministic key across equivalent inputs; key-stability unit test.
- [ ] No behavior change to slot results.

Risk: a wrong key could serve stale/incorrect slots — cover with a test asserting distinct inputs → distinct keys. Rollback: revert. Interview story potential: "I improved cache hit-rate by fixing a non-canonical cache key." (Performance STAR.)

---

## Ticket 5: Add SSRF protection to webhook delivery

Difficulty: Medium-Hard — 3 days. Layers: webhook send + validation + test.

Story: As an operator, I don't want webhook delivery to reach internal/private hosts.

Design notes: `sendPayload.ts:312-320` fetches a user-supplied URL (`redirect:"manual"` already set). Add host allow-listing / private-range + loopback blocking, applied at subscription time and send time. Consider DNS-rebind (resolve then pin).

Acceptance criteria:
- [ ] Reject private/loopback/link-local URLs; test covers each.
- [ ] Legitimate public URLs unaffected.

Risk: over-blocking legit subscribers — allow a config override for self-hosters. Rollback: flag. Interview story potential: "I hardened webhook delivery against SSRF." (Security STAR.)

---

## Ticket 6: Graceful "slot just taken" booking conflict UX

Difficulty: Medium — 2-3 days. Layers: service error + UI + i18n + test.

Story: As a booker, when my chosen slot was taken between viewing and submitting, I want a clear message and a refreshed slot list, not a generic error.

Design notes: the server-side availability re-check (`ensureAvailableUsers`) rejects the booking. Ensure it produces a specific, catchable error (e.g. a 409 conflict) that the Booker maps to a friendly message + slot refresh.

Acceptance criteria:
- [ ] Conflict returns a distinct code; Booker shows an i18n message and refetches slots.
- [ ] Test for the conflict path.

Risk: low; behavior-additive. Rollback: revert UI mapping. Interview story potential: "I turned a confusing race failure into a graceful UX." (Product+concurrency STAR.)

---

## Ticket 7: Add pagination to an unbounded booking list query

Difficulty: Medium — 2 days. Layers: repository + API + UI.

Story: As a user with many bookings, I want the list to page instead of loading everything.

Design notes: find a `findMany` without `take` on `Booking`; add cursor pagination (the repo already has `useBookingCursor`/`useBookingListData` hooks — follow them). Use the `[userId, ...]` indexes.

Acceptance criteria:
- [ ] Cursor-based pagination; stable ordering; test.

Risk: ordering/cursor bugs → duplicates or gaps; test boundaries. Rollback: revert. Interview story potential: "I paginated an unbounded query and cut page load." (Performance STAR.)

---

## Ticket 8: Dead-letter + alert for exhausted tasks

Difficulty: Medium-Hard — 3 days. Layers: tasker + observability.

Story: As an operator, I want to be alerted when a task exhausts retries.

Design notes: `task-processor.ts` retries to `maxAttempts` then stops silently. Add a terminal "failed" state (or a query + alert) and log/emit a metric. Don't change retry semantics, just add visibility.

Acceptance criteria:
- [ ] Exhausted tasks are discoverable + alert-able; test the terminal transition.

Risk: low. Rollback: revert. Interview story potential: "I added dead-letter visibility to the job queue." (Reliability/observability STAR.)

---

## Ticket 9: Enforce webhook signature presence

Difficulty: Medium — 2 days. Layers: webhook + validation.

Story: As a security-conscious operator, I want to prevent unsigned webhook delivery.

Design notes: `sendPayload.ts:299` falls back to `"no-secret-provided"`. Require a secret on webhook creation (or warn+block on send), with a migration/backfill plan for existing secretless webhooks.

Acceptance criteria:
- [ ] New webhooks require a secret; existing handled gracefully; test.

Risk: breaking existing secretless webhooks — backfill/generate secrets. Rollback: flag. Interview story potential: "I closed an unsigned-webhook gap with a safe migration."

---

## Ticket 10: Resolve the reschedule minimum-notice edge cases

Difficulty: Medium — 2-3 days. Layers: service + test.

Story: As a host, I want the minimum-reschedule-notice rule enforced consistently.

Design notes: `validateRescheduleRestrictions` (`RegularBookingService.ts:435-484`) checks notice for non-organizers and swallows some errors deliberately. Audit the swallowed-error path (`:476-483`) and add explicit tests for organizer vs non-organizer, within/outside notice.

Acceptance criteria:
- [ ] Table-driven tests for the notice matrix; the swallow is documented/justified.

Risk: changing swallow behavior could surface previously-hidden errors — test carefully. Rollback: revert. Interview story potential: "I clarified reschedule-notice enforcement and its error handling."

---

## Ticket 11: Composite rate-limit key for booking

Difficulty: Medium — 2 days. Layers: API + lib.

Story: As an operator, I don't want all users behind one NAT throttled together.

Design notes: `event.ts:37-40` keys on hashed IP only. Consider a composite key (IP + eventTypeId) where appropriate, balancing abuse protection vs. false throttling. Measure/reason about the tradeoff.

Acceptance criteria:
- [ ] Documented key change; test; no regression in abuse protection.

Risk: weakening rate limiting → abuse; reason explicitly and keep a global cap. Rollback: revert. Interview story potential: "I refined rate-limit keying to reduce false positives without weakening abuse protection."

---

## Ticket 12: Add integration test for cross-tenant event-type isolation

Difficulty: Medium — 2 days. Layers: integration tests.

Story: As a maintainer, I want proof that users can't read/mutate other tenants' event types.

Design notes: build two tenants; assert every event-type mutation rejects cross-tenant access. This both documents the contract and would catch the PBAC regression (pair with Ticket 1).

Acceptance criteria:
- [ ] Cross-tenant read/update/delete all rejected; runs in `integration` mode.

Risk: none (tests). Interview story potential: "I locked down tenant isolation with integration tests." (Security/testing STAR.)

---

## Working method

For each: write a one-page design note (problem, approach, alternatives, risk, rollback, test plan) *before* coding — that's the mid-level discipline and exactly what an RFC round tests ([07/02](../07-career-and-collaboration/02-writing-prs-and-rfcs.md)). Split by layer into small PRs. Tickets 1, 3, and 5 are the strongest interview stories.
