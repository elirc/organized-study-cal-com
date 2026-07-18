# Debugging & Code Review Rounds (Timed Simulations)

Convert the earlier scenarios into interview-format rounds: a time box, interviewer follow-ups, and a grading rubric. Do them out loud, on a timer. The skill being tested is *how you think*, narrated.

---

## Debugging Round A: intermittent double-booking (20 min)

Setup (interviewer): "Users occasionally report two bookings for the same slot. Sometimes the second booking fails with an error; sometimes both succeed. Find the cause."

Your narration should: (1) reproduce — two clients within the 2s cache window; (2) narrow — is the failure the availability re-check, the DB constraint, or a 500? (3) hypothesize — stale cache vs. true race; (4) test cheaply — log the availability inputs, inspect `Booking` rows; (5) root cause; (6) regression test.

Interviewer follow-ups:
- "Why do both sometimes succeed?" → the soft hold + re-check don't fully serialize; distinct attendees aren't blocked by the idempotency key.
- "How would you guarantee no double-booking?" → transaction with conflict predicate / `FOR UPDATE` / exclusion constraint; accept the latency.
- "Where's the 2-second window come from?" → `slots/util.ts:79` cache TTL.

Rubric — **Basic:** blames "a race" vaguely. **Solid:** distinguishes advisory (cache/hold) from guarantee (constraint), proposes a test. **Strong:** names all three layers, identifies which fails, proposes the transactional fix with its cost, and writes the regression test.

Anchors: [Flow 1](../01-codebase-cartography/05-key-flows.md), [debugging scenario 1](../05-quality-engineering/03-systematic-debugging.md).

---

## Debugging Round B: a team member deleted an event they shouldn't (15 min)

Setup: "A non-admin team member deleted a team event type. The endpoint is supposed to be ADMIN/OWNER only. Why did it work?"

Narration: trace `_router.ts:104-109` → `util.ts:159-167` → `util.ts:15-20`; find the stub returning `true`. Test cheaply: log `hasPermission`. Root cause: PBAC stub from EE removal. Fix: real role check, deny by default. Regression: failing-first non-admin delete test.

Follow-ups:
- "Why does it *look* enforced?" → named `PbacProcedure`, passes a permission string.
- "What's the general lesson?" → read the implementation, not the name; authz bugs hide behind reassuring APIs.
- "Blast radius?" → all team event-type mutations built on this procedure.

Rubric — **Strong:** finds the stub by reading the implementation, writes the failing test, proposes deny-by-default, and frames it as a class of bug (trust the impl, not the name).

Anchors: [Flow 7](../01-codebase-cartography/05-key-flows.md), [risk register #1](../09-reference/risk-register.md).

---

## Debugging Round C: webhook fires twice (15 min)

Setup: "A subscriber logs two `BOOKING_CREATED` deliveries for one booking. Debug it."

Narration: is it duplicate scheduling (two `Task` rows) or duplicate delivery (a retry)? Check `Task` rows for the booking; check the receiver's response codes and the task `attempts`. Root cause: at-least-once retry on a slow/failing receiver, or a task created without `referenceUid`. Fix: ensure `referenceUid` dedupe + idempotent consumer.

Follow-ups:
- "Can you make delivery exactly-once?" → no; make consumption idempotent.
- "How does the receiver know it's a duplicate?" → an idempotency id in the payload + the signature.

Rubric — **Strong:** separates scheduling vs. delivery, names `referenceUid` and at-least-once, and puts the fix on the consumer side.

Anchor: [debugging scenario 4](../05-quality-engineering/03-systematic-debugging.md).

---

## Review Round D: the credential-leak PR (10 min)

Setup: review a diff that changes a query from `select: { id, appId }` to `include: { app: true }` and returns it to the client.

Expected: **Blocking** — leaks `credential.key` (secret). Write the comment. Interviewer: "How would you prevent this class of bug repo-wide?" → lint rule / repository methods that never select secrets / a review checklist item.

Rubric — **Strong:** catches the leak, cites `AGENTS.md`, and proposes a systemic prevention.

Anchor: [review kata 2](../04-code-reading-gym/04-review-katas.md).

---

## Review Round E: side-effect-in-transaction PR (10 min)

Setup: review a diff adding `await sendConfirmationEmail(booking)` inside the booking `$transaction`.

Expected: **Blocking** — a slow/failing email rolls back a valid booking. Comment: commit first, enqueue via tasker. Interviewer: "What if they need the email to be guaranteed?" → transactional outbox (write the task row in the txn, relay dispatches).

Rubric — **Strong:** catches the coupling, proposes the tasker fix, and knows the outbox as the "guaranteed" answer.

Anchor: [review kata 3](../04-code-reading-gym/04-review-katas.md), [side effects](../03-architecture-and-patterns/04-side-effects-async-and-reliability.md).

---

## Review Round F: follow-webhook-redirects PR (10 min)

Setup: review a diff changing `redirect: "manual"` to `"follow"`.

Expected: **Important (security)** — SSRF surface widens on a user-supplied URL. Comment: keep manual; register final URLs. Interviewer: "What else would you add?" → private-range/DNS-rebind blocking, host allow-list.

Rubric — **Strong:** identifies SSRF, keeps manual, and lists the broader hardening.

Anchor: [review kata 8](../04-code-reading-gym/04-review-katas.md).

---

## How to run these

Do one debugging + one review round per sitting, on a timer, narrating aloud (record yourself). Grade against the rubric. The meta-skill: **narrate a method** (reproduce→narrow→hypothesize→test→fix→regress for debugging; the five layers for review), not a lucky guess. If you jump straight to an answer without narrating the search, you'll read as junior even when right.
