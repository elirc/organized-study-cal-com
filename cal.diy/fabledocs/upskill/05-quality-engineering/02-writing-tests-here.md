# Writing Tests Here

Seven recipes using the repo's real helpers. Snippets are illustrative sketches of the harness shape (labeled) — always copy patterns from an existing neighbor test, not from memory. The scenario helpers live in `@calcom/testing/lib/bookingScenario/`.

Commands: `yarn test <file>` (unit), `VITEST_MODE=integration yarn test <file>` (DB), `TZ=<zone> VITEST_MODE=timezone yarn test <file>`.

---

## Recipe 1: Happy-path booking (unit)

Model file: `packages/features/bookings/lib/handleNewBooking/test/fresh-booking.test.ts`.

```ts
// Illustrative fake code: not from this repo — shows the harness shape
const organizer = getOrganizer({ schedules: [TestData.schedules.IstWorkHours] });
await createBookingScenario(getScenarioData({ eventTypes: [{ id: 1, length: 30, users: [{ id: 101 }] }], organizer }));
mockCalendarToHaveNoBusySlots("googlecalendar");
const handler = getNewBookingHandler();
const booking = await handler(getMockRequestDataForBooking({ /* eventTypeId, start, responses */ }));
expect(booking).toEqual(expect.objectContaining({ status: "ACCEPTED" }));
```
Point: the *world* is built by `getScenarioData`; you assert the outcome.

---

## Recipe 2: Validation failure (unit)

Model: `booking-validations.test.ts`.

Build an event type with a required booking field, submit a body omitting it, expect a thrown validation error. Assert *which* field failed if the error exposes it. This exercises the dynamic schema (`getBookingData.ts:96`).

```ts
// Illustrative fake code: not from this repo
await expect(handler(getMockRequestDataForBooking({ responses: {} /* missing required */ })))
  .rejects.toThrow(/required/i);
```

---

## Recipe 3: Permission failure (unit)

Model: the cancel guard chain (`handleCancelBooking.ts:162-176`).

Set up a seated event and a caller who is not the host; expect `HttpError` 401. Table-drive the guards (already-cancelled → 400, ended → 400, non-host → 401).

```ts
// Illustrative fake code: not from this repo
await expect(cancel({ bookingData, userId: strangerId }))
  .rejects.toMatchObject({ statusCode: 401 });
```

---

## Recipe 4: Cross-tenant rejection (integration)

Model: `getEventTypesFromGroup.integration-test.ts` (real DB).

Create two users/teams; have user A try to read/mutate user B's event type; expect `FORBIDDEN`. **This is also the failing-first test for the PBAC stub** ([risk register](../09-reference/risk-register.md) #1): as a non-admin team member, call `eventType.delete` on a team event and expect `FORBIDDEN` — it fails today.

```ts
// Illustrative fake code: not from this repo
await expect(caller.viewer.eventTypes.delete({ id: teamEvent.id }))
  .rejects.toMatchObject({ code: "FORBIDDEN" }); // fails until PBAC is implemented
```
Run: `VITEST_MODE=integration yarn test`.

---

## Recipe 5: Async side effect (unit)

Model: `webhook-producer-booking-requested.test.ts`, `post-booking-handling.test.ts`.

Assert that creating a booking enqueues the expected task / calls the webhook producer, without actually sending. Mock the tasker/producer and assert it was called with the right `type` and a `referenceUid`.

```ts
// Illustrative fake code: not from this repo
expect(taskerCreateSpy).toHaveBeenCalledWith("webhookDelivery", expect.anything(), expect.objectContaining({ referenceUid: expect.any(String) }));
```

---

## Recipe 6: Idempotency invariant (unit)

Target: `booking-idempotency-key.ts`.

Assert an ACCEPTED create gets a key derived from (start, end, userId), and a cancel/reject update nulls it. Cover `create`, `update`, `updateMany` branches.

```ts
// Illustrative fake code: not from this repo
expect(generateIdempotencyKey({ startTime, endTime, userId: 1 }))
  .toEqual(generateIdempotencyKey({ startTime, endTime, userId: 1 })); // deterministic
```

---

## Recipe 7: Timezone / DST correctness (timezone mode)

Model: `complex-schedules.test.ts`, `date-overrides.test.ts`.

Assert slots on a DST-transition day map to correct wall-clock times for a non-UTC schedule (`Schedule.timeZone`). Requires `TZ` set.

```bash
TZ=America/New_York VITEST_MODE=timezone yarn test complex-schedules
```
Assert the generated slot boundaries equal the expected local times across the transition — the class of bug that "works in UTC" hides.

---

## Checklist for a good test here

- [ ] Uses `getScenarioData`/`getOrganizer`/`getBooker` builders, not hand-rolled fixtures.
- [ ] Mocks calendars/email/tasker — no network.
- [ ] Fakes the clock for time-based logic.
- [ ] Runs in the right mode (unit/integration/timezone).
- [ ] Asserts behavior/outcome, not internal call order (unless the effect *is* the behavior).
- [ ] For a bug fix: fails before the fix, passes after.

Drill: write Recipe 4 for real (the PBAC failing-first test). It's a test, a ticket, and a STAR story in one — see [06-contribution-practice/02](../06-contribution-practice/02-mid-level-feature-tickets.md).
