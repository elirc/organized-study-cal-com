# Good First Tickets

Eighteen junior-scoped tickets spread across the repo: UI, API, persistence, validation, tests, and docs. Each has a small blast radius, follows an existing pattern, and is testable. Anchors verified 2026-07-11 — re-open before starting; the codebase moves.

Per `AGENTS.md`: keep PRs under ~500 lines / 10 code files, use conventional commits (`feat:`/`fix:`/`refactor:`), open as draft, run `yarn type-check:ci --force` and `yarn test <file>` before concluding. Add UI strings to `packages/i18n/locales/en/common.json`, never inline.

Difficulty here means "junior" — locate, make a contained change, test it. "Interview story potential" tells you the STAR story shipping it produces.

---

## Ticket 1: Add a unit test for `checkForConflicts`

Difficulty: Easy — 1-2 hrs. Skills: reading algorithms, Vitest.

Story: As a maintainer, I want the interval-overlap function covered so a refactor can't silently break conflict detection.

Why good: `checkForConflicts` (`packages/features/bookings/lib/conflictChecker/checkForConflicts.ts`, ~50 lines) is pure, self-contained, and central to availability — ideal first test. Small blast radius (test-only).

Acceptance criteria:
- [ ] Cases: empty busy list → false; slot before all busy → false; exact overlap → true; adjacent (end == start) → false; seated event returns false when a matching seat exists.
- [ ] `yarn test checkForConflicts` green.

Read first: `checkForConflicts.ts` (whole file). Files touched: new `checkForConflicts.test.ts` beside it.

Implementation plan: import the fn, build `busy` arrays of `{start,end}` ISO strings, assert per case. Use `TZ=UTC` (the runner sets it).

What could go wrong: off-by-one on adjacency (the code `break`s when `busyTime.start >= slotEnd` and `continue`s when `busyTime.end <= slotStart`) — test both edges. Review questions: did you cover the `currentSeats` branch (lines 24-27)?

Interview story potential: "I added regression coverage to an untested core algorithm and found the adjacency edge case."

---

## Ticket 2: Fix the misleading error in `reserveSlot` upsert catch

Difficulty: Easy — 1 hr. Skills: error handling.

Story: As an on-call engineer, I want the slot-reservation failure to report the real cause, not "Event type not found."

Why good: `reserveSlot.handler.ts:104-109` catches any upsert error and rethrows `NOT_FOUND "Event type not found"` — misleading during incidents. Tiny, obviously-correct change.

Acceptance criteria:
- [ ] The catch preserves/logs the original error and throws a `TRPCError` with an accurate code/message (e.g. `INTERNAL_SERVER_ERROR`, "Failed to reserve slot").
- [ ] Existing `reserveSlot.handler.test.ts` still passes; add a case asserting the new message.

Read first: `reserveSlot.handler.ts:77-110`. Files touched: that handler + its test.

What could go wrong: don't leak internal error details to the client — log the original server-side, return a safe message. Review questions: is the error code semantically right (500, not 404)?

Interview story potential: "I improved incident diagnostics by fixing an error path that masked the real failure." (Also see [08-interview-prep/01-js-ts-node-deep-dive.md](../08-interview-prep/01-js-ts-node-deep-dive.md) Q10.)

---

## Ticket 3: Remove the duplicate DI module load

Difficulty: Easy — 30 min. Skills: reading DI wiring.

Story: As a maintainer, I want the container wiring to load each module once.

Why good: `di/containers/AvailableSlots.ts:36-37` loads `busyTimesModule` twice. Harmless but confusing; deleting one line is a clean, verifiable fix.

Acceptance criteria:
- [ ] One `container.load(DI_TOKENS.BUSY_TIMES_SERVICE_MODULE, busyTimesModule)` remains.
- [ ] `yarn type-check` passes; slots still resolve (run a slots-related test).

Read first: `AvailableSlots.ts:22-44`. What could go wrong: confirm the two lines are truly identical (same token + module) before deleting — they are, but verify. Review questions: any test exercising the container?

Interview story potential: "Small correctness cleanup in dependency-injection wiring."

---

## Ticket 4: Fix or document the magic-link `maxAge` comment mismatch

Difficulty: Easy — 30 min. Skills: reading auth config.

Story: As a developer, I want the magic-link expiry code and comment to agree.

Why good: `next-auth-options.ts` (~line 362) sets `maxAge: 10 * 60 * 60` (10 hours) with the comment "Magic links are valid for 10 min only." One is wrong. Determine intent (likely the comment is stale or the value should be `10 * 60`), fix the mismatch, and note it in the PR.

Acceptance criteria:
- [ ] Comment and value agree; PR description states which was authoritative and why.

Read first: the `EmailProvider` block in `next-auth-options.ts` (grep `maxAge`). What could go wrong: this is security-relevant — a 10-hour magic link is a real exposure window; do NOT just change the value without justifying; if unsure, open as a *question* PR/issue rather than guessing. Review questions: does any test or docs reference the expected expiry?

Interview story potential: "I caught a security-relevant comment/code mismatch in auth and resolved it deliberately." (STAR: ambiguity + judgment.)

---

## Ticket 5: Convert an `include` to a `select` in a query

Difficulty: Easy — 1 hr. Skills: Prisma, least-privilege.

Story: As a maintainer, I want hot queries to fetch only needed fields.

Why good: `AGENTS.md` mandates `select` over `include`. Find one `include` in a frequently-run query and narrow it, following the pattern in `reserveSlot.handler.ts:31-33`.

Acceptance criteria:
- [ ] The converted query selects exactly the fields the caller uses; nothing that consumes the result breaks (type-check will catch omissions).
- [ ] No secret fields (`credential.key`) selected.

Read first: grep `include:` under `packages/trpc/server/routers/viewer/`. What could go wrong: omitting a field the caller needs → type error (good) or runtime undefined (bad if `any` is involved). Review questions: measurable payload reduction?

Interview story potential: "I reduced over-fetching and closed a potential data-exposure by scoping a query." (Interview: [08-interview-prep/03](../08-interview-prep/03-api-and-data-modeling-questions.md).)

---

## Ticket 6: Add a test for the cancel guard chain

Difficulty: Medium — 2-3 hrs. Skills: testing guard logic.

Story: As a maintainer, I want each cancellation guard covered so authz can't regress.

Why good: `handleCancelBooking.ts:122-176` has 5+ ordered guards; a table-driven test documents the contract.

Acceptance criteria:
- [ ] Cases: already-cancelled → 400; disallowed event type → 400; missing reason when required → 400; already-ended → 400; seated non-host → 401.
- [ ] Green.

Read first: `handleCancelBooking.ts:80-176`. Files touched: a cancel test file. What could go wrong: mocking `getBookingToDelete` shape correctly. Review questions: are you asserting status codes, not just that it throws?

Interview story potential: "I documented an authorization contract with tests." (Pairs with review-kata practice.)

---

## Ticket 7: Add missing DB index justification / an index for a common filter

Difficulty: Medium — 2 hrs (design-first). Skills: schema, migrations.

Story: As a maintainer, I want frequent booking queries backed by indexes.

Why good: `Booking` already has targeted composite indexes (`schema.prisma:918-929`). Find a `prisma.booking.findMany` whose `where` isn't covered and propose (with a migration) an index — or, safer for a first PR, write a doc explaining which query each existing index serves.

Acceptance criteria:
- [ ] Either a migration adding one justified index (with EXPLAIN reasoning in the PR) OR a doc mapping indexes→queries. Do NOT add speculative indexes.

Read first: `schema.prisma:918-929`; grep `booking.findMany`. What could go wrong: indexes cost write throughput and storage — justify with a real query, or a maintainer will reject it. Review questions: does an existing index already cover it by prefix?

Interview story potential: "I connected the booking indexes to their real query patterns." (Interview: indexing tradeoffs.)

---

## Ticket 8: Extract a magic number into a named constant

Difficulty: Easy — 30 min. Skills: readability.

Story: As a reader, I want `DEFAULT_SLOTS_CACHE_TTL = 2000` and similar literals named and documented.

Why good: `slots/util.ts:79` already names the TTL — find a sibling magic number (e.g. an inline timeout/limit) and give it a named constant + comment explaining *why* that value.

Acceptance criteria:
- [ ] One literal replaced by a named const; comment explains the choice; no behavior change.

Read first: `slots/util.ts:78-120`. What could go wrong: changing the value instead of just naming it. Review questions: is the name meaningful and the "why" comment real (not restating the number)?

Interview story potential: "Readability cleanup that made a tuning parameter discoverable."

---

## Ticket 9: Add an i18n string for a hardcoded UI label

Difficulty: Easy — 1 hr. Skills: i18n.

Story: As a non-English user, I want all labels translated.

Why good: `AGENTS.md` requires UI strings go through `packages/i18n/locales/en/common.json`. (That file was filtered out of this study export, per the repo-root `SECURITY_FILTERED_FILES.md`; do this ticket in a full Cal.diy checkout.) Find a hardcoded user-facing string in a component and route it through the translation hook.

Acceptance criteria:
- [ ] String added to `en/common.json`; component uses `t("key")`; renders unchanged in English.

Read first: an existing `t("...")` usage in a Booker component for the pattern. What could go wrong: key naming collisions; missing the pluralization case. Review questions: is the key namespaced sensibly?

Interview story potential: "I closed an i18n gap." (Small but shows attention to the codebase's conventions.)

---

## Ticket 10: Write a trace-table doc for the reschedule flow

Difficulty: Medium — 2 hrs. Skills: code reading, docs.

Story: As a new contributor, I want the reschedule path documented like the other key flows.

Why good: reschedule logic is scattered (`RegularBookingService.ts:435-484` validation, `originalRescheduledBookingUtils.ts`, `EventManager.reschedule` at `EventManager.ts:615`). A doc PR is a safe, high-value first contribution.

Acceptance criteria:
- [ ] A markdown trace table (step/owner/file/what/risk) matching the style of [01-codebase-cartography/05-key-flows.md](../01-codebase-cartography/05-key-flows.md).

Read first: the three files above. What could go wrong: claiming behavior you didn't verify — anchor every row. Review questions: does each row cite a real line?

Interview story potential: "I onboarded myself by mapping the reschedule flow and left docs for the next person." (STAR: learning a complex codebase.)

---

## Ticket 11: Add validation test for `getBookingData` rejecting a missing required field

Difficulty: Medium — 2 hrs. Skills: zod, schema-driven validation.

Story: As a maintainer, I want proof that a booking missing a required custom field is rejected.

Why good: builds on the existing `getBookingData.test.ts`; teaches the dynamic-schema pattern (`RegularBookingService.ts:540-549`).

Acceptance criteria:
- [ ] A test constructs an event type with a required booking field, submits a body omitting it, asserts a validation error.

Read first: `getBookingData.ts:96`, `getBookingData.test.ts`. What could go wrong: building the `eventType.bookingFields` fixture correctly. Review questions: does it assert *which* field failed?

Interview story potential: "I tested dynamic per-event-type input validation." (Interview: zod boundaries.)

---

## Ticket 12: Guard the webhook no-secret fallback

Difficulty: Medium — 2 hrs (discuss first). Skills: security, webhooks.

Story: As a webhook receiver, I want to never receive an unsigned payload silently.

Why good: `sendPayload.ts:296-299` returns `"no-secret-provided"` as the signature when no secret exists. Add a log/warning (and consider a doc note) so unsigned deliveries are visible. Keep it minimal — behavior-preserving observability first.

Acceptance criteria:
- [ ] When `secret` is absent, a warning is logged with the webhook id; delivery still works; `sendPayload.test.ts` covers it.

Read first: `sendPayload.ts:296-327`. What could go wrong: don't change delivery behavior in a first PR (that's a bigger design change — see [06-contribution-practice/02](02-mid-level-feature-tickets.md)); just add visibility. Review questions: does the log leak the payload/secret? (It must not.)

Interview story potential: "I added observability around an unsigned-webhook edge case."

---

## Ticket 13: Resolve a `TODO: Inject.` by threading a dependency

Difficulty: Medium — 2-3 hrs. Skills: DI, testability.

Story: As a maintainer, I want `originalRescheduledBookingUtils.ts` to receive its dependency instead of importing a singleton.

Why good: `originalRescheduledBookingUtils.ts:7` literally says `// TODO: Inject.` — follow the repo's DI/repository-injection pattern to make the function testable.

Acceptance criteria:
- [ ] The dependency is passed in (with a default for existing callers) so a test can inject a fake; existing callers unchanged.

Read first: the file + how `RegularBookingService` receives `IBookingServiceDependencies` (`:423-433`). What could go wrong: changing every call site (keep a default to bound the diff). Review questions: is the injected type an interface, not a concrete class?

Interview story potential: "I improved testability by inverting a hidden dependency." (STAR: technical tradeoff.)

---

## Ticket 14: Add a test asserting `idempotencyKey` is set on create and cleared on cancel

Difficulty: Medium — 2 hrs. Skills: Prisma extensions, invariants.

Story: As a maintainer, I want the idempotency invariant covered.

Why good: `booking-idempotency-key.ts` is pure logic; a test locks in "ACCEPTED create → key set; cancel/reject → key null."

Acceptance criteria:
- [ ] Test exercises the extension's create/update/updateMany branches.

Read first: `booking-idempotency-key.ts` (whole file). What could go wrong: testing an extension in isolation (call `generateIdempotencyKey` semantics via the exported behavior, or mock `query`). Review questions: did you cover `updateMany`?

Interview story potential: "I tested a cross-cutting idempotency invariant enforced via a Prisma extension." (Strong system-design signal.)

---

## Ticket 15: Add ARIA/label to a Booker interactive element

Difficulty: Easy — 1-2 hrs. Skills: accessibility.

Story: As a screen-reader user, I want booking controls labeled.

Why good: accessibility gaps are common junior wins; the Booker has date/time pickers (`apps/web/modules/bookings/components/`). Find one control missing an accessible name and add it.

Acceptance criteria:
- [ ] Control has an accessible label; verified with devtools accessibility pane; no visual change.

Read first: the component + any existing `aria-*` usage. What could go wrong: duplicating a visible label (use `aria-label` vs `aria-labelledby` correctly). Review questions: does it read sensibly in a screen reader?

Interview story potential: "I improved booking-flow accessibility." (Interview: a11y basics.)

---

## Ticket 16: Add a "no available slots" empty-state message check test

Difficulty: Medium — 2 hrs. Skills: React testing / hooks.

Story: As a booker, I want a clear message when a host has no availability.

Why good: `useNonEmptyScheduleDays` / `HavingTroubleFindingTime` already exist; add a test (or improve the message via i18n) for the empty state.

Acceptance criteria:
- [ ] Test renders the Booker with an empty schedule and asserts the empty-state UI; or the message is improved and i18n-routed.

Read first: `Booker.tsx` empty-state branch + `useNonEmptyScheduleDays`. What could go wrong: over-mocking; test the observable behavior. Review questions: is the message translated?

Interview story potential: "I covered an empty-state UX path with a test."

---

## Ticket 17: Document the tasker task lifecycle in `packages/features/tasker/README.md`

Difficulty: Easy — 1-2 hrs. Skills: async systems, docs.

Story: As a contributor, I want the task states and retry rules written down.

Why good: `task-processor.ts:11-40` + the `Task` model (`schema.prisma:1825-1850`) define a lifecycle worth documenting: created → attempted → succeeded/retried → (max attempts). A README exists to extend.

Acceptance criteria:
- [ ] Doc covers states, `maxAttempts`, `referenceUid` dedupe, internal-vs-redis backends, and "handlers must be idempotent."

Read first: `tasker.ts`, `task-processor.ts`, existing `tasker/README.md`. What could go wrong: describing behavior you didn't read (anchor it). Review questions: does it warn about at-least-once delivery?

Interview story potential: "I documented the background-job lifecycle." (Interview: reliability patterns.)

---

## Ticket 18: Add a regression test for the `MINUTES_TO_BOOK` reservation expiry

Difficulty: Medium — 2 hrs. Skills: time, testing.

Story: As a maintainer, I want the hold's `releaseAt` computation covered.

Why good: `reserveSlot.handler.ts:29` computes `releaseAt = now + MINUTES_TO_BOOK`; a test with a fixed clock locks the expiry math.

Acceptance criteria:
- [ ] Test asserts `releaseAt` equals `slotStart-independent now + MINUTES_TO_BOOK` using a mocked clock; runner is `TZ=UTC`.

Read first: `reserveSlot.handler.ts:26-29`, `MINUTES_TO_BOOK` in `packages/lib/constants.ts` (line 137; defaults to `"5"`). What could go wrong: timezone/DST flakiness — use UTC and fake timers. Review questions: does it avoid real `Date.now()`?

Interview story potential: "I made a time-based feature deterministically testable." (Interview: testing time/randomness — see [05-quality-engineering/01-testing-strategy.md](../05-quality-engineering/01-testing-strategy.md).)

---

## How to pick

Start with Ticket 1 or 3 (pure, tiny, obviously correct). Then a test ticket (6, 11, 14) to learn the harness. Save 4, 7, and 12 until you've read [05-quality-engineering/05-security-checklist.md](../05-quality-engineering/05-security-checklist.md) — they touch security/perf and need judgment, not just mechanics. For each, write the PR description using [07-career-and-collaboration/02-writing-prs-and-rfcs.md](../07-career-and-collaboration/02-writing-prs-and-rfcs.md).
