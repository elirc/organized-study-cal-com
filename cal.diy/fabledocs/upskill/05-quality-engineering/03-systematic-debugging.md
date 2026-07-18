# Systematic Debugging

The method, then five repo-specific scenarios. The method is the transferable skill; the scenarios teach you where Cal.diy's seams are.

## The method

1. **Reproduce** — get a reliable, minimal repro. A bug you can't trigger on demand, you can't fix with confidence.
2. **Narrow** — binary-search the layers. Is it client or server? Read path or write path? Before or after the DB write? Each answer halves the search space.
3. **Hypothesize** — form one falsifiable guess ("the availability re-check passes because the cache is stale").
4. **Test cheaply** — prove/disprove with the least effort: a log line, a `console.info` (the tasker uses these), a query in `yarn db-studio`, a single test.
5. **Fix the root cause** — not the symptom. If two requests race, add a constraint, don't add a `sleep`.
6. **Add regression coverage** — a test that fails before your fix and passes after. This is the step juniors skip and interviewers probe.

Tools available here: React/Chrome devtools (client), the Node debugger / `console.info` (the tasker logs liberally, `task-processor.ts:13-39`), Prisma query logging (`DEBUG` / Prisma `log`), `yarn db-studio` (inspect rows), Sentry (`withReporting` captures with `functionName` + redacted args, `sentryWrapper.ts:236`), CI logs, and Playwright traces (`apps/web/test-results/`).

---

## Scenario 1: A slot shows as available but booking it fails with "no longer available"

Reproduction: two browsers open the same booking page; both pick the same slot; the second submit fails after the first books.

First question: is this the *expected* double-booking guard firing, or a stale-cache false negative? (Distinguish "working as designed" from "bug".)

Narrowing path:
1. Confirm the slot list came from cache — availability caches 2s in Redis (`slots/util.ts:104-120`, TTL `:79`). If both loads were within 2s, both saw the same "free" slot. Expected.
2. Check where the failure is thrown: server-side availability re-check (`ensureAvailableUsers`) or the DB unique constraint (`Booking.idempotencyKey`, `schema.prisma:855`)?
3. If it's the re-check: correct behavior — the server refused to double-book. If it's a *500* (not a clean "unavailable" message): that's the bug — the guard should produce a user-friendly conflict, not a crash.

Useful probes: log the slot's availability computation inputs; inspect `Booking` rows in db-studio for the contested time.

Likely root causes: (a) working as designed but bad UX (no graceful "someone just took this" message); (b) the re-check reads stale busy times; (c) a 500 leaking instead of a 4xx.

Regression test: a test where two `createBooking` calls target the same slot; assert the second gets a clean conflict, not an unhandled error.

Senior lesson: "available" is advisory (cache + soft hold); the *guarantee* is the constraint. Debugging starts by deciding whether you're looking at the advisory or the guarantee.

Interview version: "Tell me how you'd debug an intermittent double-booking report." → reproduce with two clients within the cache window; narrow to re-check vs. constraint; the fix is UX (graceful conflict) + possibly a transaction for true serialization.

---

## Scenario 2: Two reservations for the same slot both succeed

Reproduction: fire two `reserveSlot` calls concurrently for the same slot with different `uid` cookies.

First question: is the collision in the check or the write? (It's the gap between them.)

Narrowing path:
1. Read `reserveSlot.handler.ts:67-103`: `findReservedByOthers` (check) then `upsert` (write), no transaction.
2. Add timing logs around both; under concurrency both checks return "free" before either writes.
3. Confirm the `SelectedSlots` unique constraint (`schema.prisma:1447`) is on `[userId, slot..., uid]` — so *different* `uid`s don't collide at the DB either. The race is real.

Useful probes: db-studio on `SelectedSlots`; log the `uid` and check result per request.

Likely root cause: TOCTOU — the check and write aren't atomic, and the unique index includes `uid`, so distinct visitors legitimately both get holds.

Regression test: hard to test true concurrency in unit tests; instead test the *fix* — e.g., that a race-safe write path rejects the second hold, or document that holds are intentionally soft.

Senior lesson: not every race needs a lock. Here holds are *meant* to be soft (advisory UX). The bug is only a bug if you expected exclusivity. Decide the intended semantics before "fixing."

Interview version: this is the cleanest "walk me through a race condition" answer in the repo — see [08-interview-prep/01](../08-interview-prep/01-js-ts-node-deep-dive.md) Q2 and [04-code-reading-gym/04-review-katas.md](../04-code-reading-gym/04-review-katas.md).

---

## Scenario 3: A team member can delete a team event type they shouldn't be able to

Reproduction: as a non-admin member of a team, call the `eventType.delete` mutation on a team event.

First question: which authz layer should stop this — the middleware or the handler?

Narrowing path:
1. `_router.ts:104-109`: `delete` uses `createEventPbacProcedure("eventType.delete", [ADMIN, OWNER])` — looks enforced.
2. `util.ts:159-167`: team branch calls `PermissionCheckService.checkPermission`.
3. `util.ts:15-20`: that class returns `true` unconditionally. **Root cause found** — the permission check is a stub (the real PBAC lived in removed enterprise code).

Useful probes: add a log in the team branch printing the `hasPermission` result; you'll see it's always `true`.

Likely root cause: EE→community fork left a permissive stub.

Regression test: a test where a non-admin team member calls a PBAC procedure on a team event and expects `FORBIDDEN` — it will *fail* today, documenting the gap; it passes once you implement a role check.

Senior lesson: authorization bugs hide behind reassuring names. Grep the *implementation*, not the call site. Fix: enforce role from `team.members` against `fallbackRoles`, deny by default.

Interview version: "Describe a security bug you found." → this, framed as: named check, opened the impl, found the stub, wrote the failing test, proposed the deny-by-default fix. Strong STAR (see [08-interview-prep/06](../08-interview-prep/06-behavioral-star-stories.md)).

---

## Scenario 4: Webhook fires twice for one booking

Reproduction: a subscriber logs two `BOOKING_CREATED` deliveries for a single booking.

First question: duplicate *scheduling* or duplicate *delivery* (a retry)?

Narrowing path:
1. Delivery is at-least-once (`task-processor.ts:23-34` retries on throw). If the receiver returned non-2xx or the handler threw *after* the HTTP POST succeeded, it retries → duplicate delivery.
2. Duplicate scheduling would show two `Task` rows; `Task @@unique([referenceUid, type])` (`schema.prisma:1846`) should prevent that *if* a `referenceUid` is passed.
3. Check whether the webhook task was created with a `referenceUid` (`tasker.ts:32-34` options). If not, dedupe isn't in effect.

Useful probes: query `Task` rows for the booking; inspect the receiver's response codes; check `attempts` on the task.

Likely root causes: (a) receiver is slow/non-idempotent and the retry re-delivers; (b) task created without `referenceUid` so scheduling dedupe is off.

Regression test: assert the webhook task is created with a `referenceUid` so a second identical schedule is a no-op.

Senior lesson: at-least-once means the *receiver* must be idempotent; you cannot make delivery exactly-once, so you make consumption idempotent. This is the reliability lesson interviewers want.

Interview version: "How do you handle duplicate webhooks?" → signature to verify origin (`sendPayload.ts:296-318`), `referenceUid` to dedupe scheduling, idempotent consumer for delivery retries.

---

## Scenario 5: Availability is wrong for a user in a non-UTC timezone (off by an hour around DST)

Reproduction: a host in a DST-observing zone; bookings appear shifted by an hour across the DST boundary.

First question: is the bug in stored data, in the availability computation, or in display?

Narrowing path:
1. `Availability.startTime/endTime` are `@db.Time` (`schema.prisma:967-968`) — wall-clock times without a date; the zone comes from `Schedule.timeZone` (`:953`).
2. The computation applies the zone when building date ranges (`buildDateRanges`, imported in `slots/util.ts:43`). A DST bug usually means a `.utc()` conversion dropped or added the offset at the boundary.
3. Reproduce deterministically: the repo has timezone-gated tests (`VITEST_MODE=timezone yarn test`, requires `TZ` set). Run them at the offending zone.

Useful probes: log the computed UTC ranges for a day spanning the DST change; compare to expected wall-clock.

Likely root cause: applying a fixed offset instead of a zone-aware conversion around the transition.

Regression test: a timezone test (gated by `VITEST_MODE=timezone`) asserting slots on the DST-change day map to the correct wall-clock times.

Senior lesson: store wall-clock + zone, convert at the edges, and test the *transition days* specifically — "it works in UTC" hides every DST bug. This is why the repo isolates timezone tests behind an env flag.

Interview version: "What's hard about scheduling across timezones?" → wall-clock vs. instant, DST transitions, storing zone separately; cite `@db.Time` + `Schedule.timeZone` as the storage choice.

---

## Practice

Pick Scenario 3 (the authz stub) and actually do it: write the failing `FORBIDDEN` test, watch it pass (bug present), then sketch the fix. That single exercise gives you a debugging demo, a security finding, a ticket ([06-contribution-practice/02](../06-contribution-practice/02-mid-level-feature-tickets.md)), and a STAR story. Then convert Scenarios 1-2 into timed interview narrations per [08-interview-prep/05-debugging-and-code-review-rounds.md](../08-interview-prep/05-debugging-and-code-review-rounds.md).
