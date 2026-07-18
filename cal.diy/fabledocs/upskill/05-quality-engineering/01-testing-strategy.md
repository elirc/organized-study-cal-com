# Testing Strategy

## The layers here

| Layer | Tool / mode | Location | Tests what |
| --- | --- | --- | --- |
| Unit | Vitest (default, `TZ=UTC`, `pool: forks`) | `*.test.ts` beside source | pure logic, guards, schema parsing |
| Integration | Vitest `VITEST_MODE=integration` | `*.integration-test.ts` | DB-backed behavior, real Prisma |
| Timezone | Vitest `VITEST_MODE=timezone` (needs `TZ`) | timezone-sensitive tests | DST/offset correctness |
| E2E | Playwright, projects per surface | `apps/web/playwright/`, `packages/app-store/`, embed | critical user flows in a real browser |

The booking engine has a rich unit-test suite in `packages/features/bookings/lib/handleNewBooking/test/` (fresh-booking, reschedule, booking-limits, booking-validations, spam, managed-event-type, dynamic-group, complex-schedules, date-overrides, ...). This is the model to follow.

## What belongs at each layer

- **Unit:** `checkForConflicts` overlap math, `getBookingData` schema rejection, cancel guard chain, the idempotency-key extension, `getAssignmentReasonCategory`. Anything pure or mockable.
- **Integration:** the actual `booking.create` transaction, cross-tenant query rejection, denormalized query behavior (`spam-booking.integration-test.ts`, `getEventTypesFromGroup.integration-test.ts`). Needs Postgres.
- **Timezone:** slot generation across a DST boundary — gated so it can't flake the default suite.
- **E2E:** the booking flow end to end, embed rendering, app-store install. Slow, high-confidence; reserve for the money paths.

## What NOT to test

- Framework/library behavior (Prisma, zod, Next) — trust it.
- Getters/trivial passthroughs.
- Exact HTML structure (brittle) — test behavior/roles instead.
- The 2,690-line service as a whole — test its seams (the `handleNewBooking/*` helpers), which is exactly what the repo does.

## Fixtures, builders, mocks, isolation

The repo has a purpose-built scenario harness (`@calcom/testing/lib/bookingScenario/`):
- `createBookingScenario(getScenarioData({...}))` builds a full world (users, event types, schedules, credentials) — see `booking-validations.test.ts:7-17`.
- `getOrganizer`, `getBooker`, `getMockBookingAttendee`, `TestData.schedules.IstWorkHours` — named builders for readable setup.
- `getMockRequestDataForBooking` — constructs a valid booking request body.
- `prismaMock` (`@calcom/testing/lib/__mocks__/prisma`) — mocked Prisma for unit tests; `setupAndTeardown` handles isolation.
- `mockCalendarToHaveNoBusySlots` — controls external calendar behavior deterministically.

The lesson: good tests read like a spec because the *builders* hide setup noise. When you add a test, reuse these — don't hand-roll fixtures.

## Time, randomness, flake prevention

- Force `TZ=UTC` for determinism (the runner does this); isolate timezone tests behind the mode.
- Mock the clock for `releaseAt`/expiry math (`reserveSlot.handler.ts:29`) — never real `Date.now()`.
- Mock external calls (calendars, email) via the scenario helpers — no network in unit tests.
- `pool: "forks"` avoids a class of RPC-during-fetch flake (`vitest.workspace.ts`).
- Randomness (uuids) — assert shape/uniqueness, not exact values, or inject a deterministic generator.

## The transferable rule

Test at the **lowest layer that gives confidence**: pure logic as unit tests (fast, many), DB/cross-entity behavior as integration (fewer), full journeys as E2E (fewest). This is the test pyramid; Cal.diy follows it, with the twist that timezone correctness gets its own mode.

Drill: for each change, name the layer — (a) fix `checkForConflicts` adjacency, (b) add a `Booking` column, (c) reject a cross-tenant event-type read, (d) DST slot bug. Answers: unit / integration / integration / timezone. See [02-writing-tests-here.md](02-writing-tests-here.md) for the recipes.
