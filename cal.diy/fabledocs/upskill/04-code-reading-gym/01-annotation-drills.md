# Annotation Drills

Ten real excerpts. For each: open the file, annotate **inputs, outputs, dependencies, invariants, side effects, failure modes** before reading the "senior notices" notes. Grade with the rubric — naming the file is Basic; tracing data + boundary is Solid; invariant + risk + test is Strong.

Do these with the file open. Don't read ahead to the answer notes.

---

## Drill 1: `checkForConflicts` (interval overlap)

File: `packages/features/bookings/lib/conflictChecker/checkForConflicts.ts` (whole file, ~50 lines).

Annotate: What are the two early returns and why? What does the `for` loop's `break` vs `continue` encode? Is the loop O(n) or O(n log n)?

Self-check:
- Inputs: `busy` (array of `{start,end}`), `time` (Dayjs), `eventLength` (minutes), optional `currentSeats`.
- Output: boolean, `true` = conflict.
- Invariants: busy times are sorted by start (line 37) *inside* the function — callers don't have to. Slot is `[slotStart, slotEnd)`.
- The `break` (line 40-42): once a busy block starts at/after the slot ends, no later block (also later-starting) can overlap → stop early. The `continue` (line 43-44): a busy block ending at/before slot start is fully before us → skip. Anything else overlaps → `true`.
- Failure modes: adjacency — a busy block ending exactly at `slotStart` is NOT a conflict (`<=`), and one starting exactly at `slotEnd` is NOT a conflict (`>=`). Get these `<`/`<=` wrong and you either double-book or hide free slots.

Senior notices: sorting happens every call (fine for small arrays, wasteful if the caller already sorted); `currentSeats` short-circuit (line 26) compares ISO strings, so a millisecond/precision mismatch would miss a seat match. Overall complexity is dominated by the sort: O(n log n).

Interview tie-in: this is a whiteboard-able "does interval A overlap any of B?" — see [08-interview-prep/01](../08-interview-prep/01-js-ts-node-deep-dive.md).

---

## Drill 2: The booking front door

File: `apps/web/pages/api/book/event.ts` (whole file, 63 lines).

Annotate: List the four defenses that run *before* any booking logic, in order. Which are conditional? Where does the anonymous-user default come from?

Self-check:
- Order: (1) Turnstile if env flag set (20-25), (2) bot detection (28-35), (3) rate limit by hashed IP (37-40), (4) session lookup (42) — booking proceeds with `userId: -1` if anonymous (53).
- Side effects: none until the service call (50); this file is pure orchestration + defenses.
- Invariant: `creationSource` is forced to `WEBAPP` (46) so downstream can't be tricked about origin.
- Failure modes: rate limit keyed only on IP hash → shared-NAT users share a bucket; bot detection is behind a feature flag (per-event-type) so it may be off.

Senior notices: the service is resolved via DI (`getRegularBookingService()`, 49) not `new`-ed — so this route and other callers share one wiring; `defaultResponder` (63) wraps the handler to standardize error→HTTP mapping (find it).

---

## Drill 3: The DI container for slots

File: `packages/features/di/containers/AvailableSlots.ts` (whole file, 45 lines).

Annotate: How many distinct modules load? Which token appears twice? What does `getAvailableSlotsService()` actually return, and where is that type defined?

Self-check:
- ~19 `container.load` calls; `DI_TOKENS.BUSY_TIMES_SERVICE_MODULE` loads twice (36-37) — a bug-shaped nit.
- `container` is a module-level singleton (22) — created once at import.
- Output: `container.get<AvailableSlotsService>(...)` (43); the type comes from `slots/util.ts`.

Senior notices: the container is eager (loads at import time), so importing this file has a cost and side effects; the duplicate load is idempotent here but is exactly how a *wrong* token would sneak in unnoticed. Invariant to state in review: "each token loaded exactly once."

---

## Drill 4: Slot reservation (concurrency)

File: `packages/trpc/server/routers/viewer/slots/reserveSlot.handler.ts` (whole file, 126 lines).

Annotate: Draw the read→write timeline. Mark the exact lines that form a TOCTOU race. What are the three independent problems in this file?

Self-check:
- Read: `findReservedByOthers` (67-75). Write: `upsert` loop (79-103). No transaction between them → race.
- Three problems: (1) TOCTOU race; (2) `FIXME` at 80-82 — team events reserve against `eventType.users` (creator only) not hosts; (3) `catch` at 104-109 rethrows "Event type not found" for any upsert error.
- Side effects: DB upserts + a `Set-Cookie` (115-122) with embed-aware SameSite.
- Invariant intended: at most one active reservation per slot per foreign `uid`; actually only *soft* — enforced by nothing atomic.

Senior notices: the cookie logic (111-122) is a genuine cross-origin embed concern (SameSite=None+Secure for iframes) — not incidental. The comment block there is worth reading as "why the code is weird."

---

## Drill 5: The cancel guard chain

File: `packages/features/bookings/lib/handleCancelBooking.ts:80-176`.

Annotate: Number each guard 1..n. For each, is it a *state* check or a *permission* check? Is the order correct?

Self-check:
- 1 already-cancelled (122), 2 no user (129), 3 event disallows cancel (133), 4 compute host (140), 5 reason required (148), 6 already-ended (155), 7 seated non-host (162-176).
- State checks (1,3,6) generally precede/interleave permission (4,7). Guard 1 exists to avoid deleting a reschedule's calendar event.
- Failure modes: host identity mixes `userId` and `email === cancelledBy` (141) — email-based authz is weaker.

Senior notices: the ordering is deliberate — you must not act on terminal state, and you check "does this event even allow cancelling" before spending effort on permission. State-before-permission is the transferable rule.

---

## Drill 6: The idempotency Prisma extension

File: `packages/prisma/extensions/booking-idempotency-key.ts` (whole file, 55 lines).

Annotate: What triggers key generation vs. clearing? What are the key's inputs? What does it deliberately exclude?

Self-check:
- Generate on `create` when status === ACCEPTED (28-35); clear on update/updateMany when status becomes CANCELLED/REJECTED (39-50).
- Key inputs: `start.end.userId(.reassignedById)` via `uuidv5` (17-20). Excludes attendee identity and event type.
- Invariant: an ACCEPTED booking always has a key; a cancelled one has none (so a new booking at the same time can reuse the "slot" without unique collision).

Senior notices: this is an *invisible* control — call sites set `idempotencyKey: null` (`RegularBookingService.ts:204`) and the extension overwrites it. Great for guarantees, dangerous for readers. In review, you'd want a comment at the call site pointing to the extension.

---

## Drill 7: The tRPC auth middleware

File: `packages/trpc/server/middlewares/sessionMiddleware.ts:6-40`.

Annotate: What does `isAuthed` add to `ctx`? What type guarantee do downstream handlers get? What does `isAdminMiddleware` add on top?

Self-check:
- `isAuthed` resolves `{user, session}` (9), throws `UNAUTHORIZED` if missing (15-16), sets Sentry user (18), and `return next({ ctx: { user, session } })` (20-22) — so `ctx.user` is now non-null *in the type system*.
- `isAdminMiddleware` pipes `isAuthed` then checks `user.role === "ADMIN"` (25-30).
- Side effect: `SentrySetUser`.
- Invariant: authentication only — NOT resource authorization.

Senior notices: after `isAuthed`, handlers skip null checks because the type narrowed; but "logged in" ≠ "allowed to touch this record" — that's per-handler (Drill 8). Confusing the two is the top authz bug class.

---

## Drill 8: Resource-scoped authorization (and the stub)

File: `packages/trpc/server/routers/viewer/eventTypes/util.ts:15-20` and `:103-199`.

Annotate: For a *personal* event, what makes authz correct? For a *team* event, trace the check to its implementation. What does it return?

Self-check:
- Personal (151-158): `event.userId === ctx.user.id` or caller in `event.users` → correct object-level check (stops IDOR).
- Team (159-167): `new PermissionCheckService().checkPermission(...)` → but that class (15-20) returns `true` unconditionally. So team authz is bypassed. Confirmed.
- Assigned-users validation (178-196) still runs and is real.

Senior notices: the procedure is *named* `PbacProcedure` and takes a permission string — it *looks* enforced. The lesson: read the implementation, never trust the name. This is a first-class finding (see [09-reference/risk-register.md](../09-reference/risk-register.md)).

---

## Drill 9: The webhook send + signature

File: `packages/features/webhooks/lib/sendPayload.ts:296-327`.

Annotate: What exactly is signed? What happens if there's no secret? Why `redirect: "manual"`?

Self-check:
- `createWebhookSignature` (296-299) HMAC-SHA256 over the exact `body` string with the webhook secret; header `X-Cal-Signature-256` (316).
- No secret → signature is the literal `"no-secret-provided"` (299) — delivery still happens, unsigned.
- `redirect: "manual"` (319) → don't follow redirects (so a malicious/misconfigured subscriber can't bounce the signed request elsewhere — SSRF hardening).
- Failure mode: signing must be over the same bytes actually sent (it is — `body` is passed to both).

Senior notices: the no-secret fallback means receivers MUST reject `"no-secret-provided"`; and the subscriber URL is attacker-influenced → this `fetch` is an SSRF surface that also wants URL allow-listing.

---

## Drill 10: The task processor retry loop

File: `packages/features/tasker/task-processor.ts:11-40`.

Annotate: What are the possible outcomes per task? How does a failure get retried? What delivery guarantee does this give?

Self-check:
- `getNextBatch` (12) → for each task, resolve handler (18-22), run it (23), on success `Task.succeed` (25), on failure `Task.retry` with `minRetryIntervalMins` (27-33).
- `Promise.allSettled` (37) so one failing task doesn't abort the batch; failed/succeeded counted (38-39).
- Guarantee: **at-least-once** — a task that half-completes then throws is retried, so handlers must be idempotent.

Senior notices: no visible dead-letter — a task failing `maxAttempts` times just stops being retried (check `Task.retry`/`getNextBatch` for the exact terminal condition); an unhandled poison payload could churn. The `allSettled` choice (vs `all`) is deliberate and correct here — contrast with `reserveSlot.handler.ts:79` using `all`.

---

## Grading rubric (apply to every drill)

- **Basic:** you named the file and roughly what it does.
- **Solid:** you traced inputs→outputs, named the dependency/boundary, and listed side effects.
- **Strong:** you stated the invariant it protects, named a concrete failure mode with the line, and said what test you'd write.

Do all ten, then take the three with confirmed issues (4, 8, 9/10) into [04-code-reading-gym/04-review-katas.md](04-review-katas.md) and write the review comment you'd leave.
