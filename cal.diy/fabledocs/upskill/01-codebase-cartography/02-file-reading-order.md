# File Reading Order

Twenty-five files, in order. Per file: why it matters, what to look for, what to ignore. Three tracks share the list — juniors stop at 15, mid-levels do all 25, seniors add the "senior detour" notes. Anchors verified 2026-07-11.

## Orientation (1-5)

1. `README.md` — what Cal.diy is (Cal.com minus enterprise). *Look for:* the "What's different" list. *Ignore:* badges.
2. `package.json` (root) — scripts + workspaces. *Look for:* `dev`, `test` (forces `TZ=UTC`), `build` filters. *Ignore:* the embed publish scripts.
3. `AGENTS.md` — the house rules that shape every file. *Look for:* `select` over `include`, error-type rules, no barrel imports, PR size. *Ignore:* nothing — this is the codebase's constitution.
4. `packages/prisma/schema.prisma:851-930` — the `Booking` model. *Look for:* `idempotencyKey`, `status`, indexes. *Ignore:* the long tail of optional columns for now.
5. `packages/prisma/schema.prisma:156-275` — `EventType`. *Look for:* `length`, `minimumBookingNotice`, `seatsPerTimeSlot`, `bookingFields`, `bookingLimits`, `schedulingType`. *Ignore:* the RR-weight and instant-meeting fields.

## The write path (6-12)

6. `apps/web/pages/api/book/event.ts` — booking front door. *Look for:* the 4 pre-checks + DI service call.
7. `packages/features/bookings/lib/service/RegularBookingService.ts:486-560` — the handler's opening. *Look for:* event-type load, reschedule validation, schema build, body parse. *Ignore:* the 2,000 lines below for now.
8. `packages/features/bookings/lib/handleNewBooking/getBookingData.ts:96` — schema-driven validation entry. *Look for:* how `bookingFields` drives the schema.
9. `packages/features/bookings/lib/handleNewBooking/createBooking.ts:60-146` — the DB write. *Look for:* the `$transaction`, the `select` shape.
10. `packages/prisma/extensions/booking-idempotency-key.ts` — cross-cutting invariant. *Look for:* when the key is set/cleared. *Senior detour:* note it overwrites call-site `null`s.
11. `packages/features/bookings/lib/handleCancelBooking.ts:80-176` — the guard chain. *Look for:* state-before-permission ordering.
12. `packages/features/bookings/repositories/BookingRepository.ts:340` — where queries live. *Look for:* `select` usage; no business logic (per `AGENTS.md`).

## The read path (13-17)

13. `packages/trpc/server/routers/viewer/slots/getSchedule.handler.ts` — 8 lines; the thin handler. *Look for:* DI resolution.
14. `packages/features/di/containers/AvailableSlots.ts` — the wiring. *Look for:* ~19 modules; the duplicate load (36-37).
15. `packages/trpc/server/routers/viewer/slots/util.ts:1-120` — the engine's imports + cache. *Look for:* the subsystem list in the imports; `withSlotsCache`; TTL 2000ms. *(Juniors can stop here.)*
16. `packages/features/bookings/lib/conflictChecker/checkForConflicts.ts` — pure overlap algorithm. *Look for:* the `break`/`continue` logic.
17. `packages/trpc/server/routers/viewer/slots/reserveSlot.handler.ts` — soft holds + a race. *Look for:* check-then-write; `FIXME`; the misleading catch.

## Auth & API surface (18-21)

18. `packages/trpc/server/trpc.ts` — tRPC init (superjson, errorFormatter). *Look for:* what `procedure`/`router` are.
19. `packages/trpc/server/middlewares/sessionMiddleware.ts:6-40` — `isAuthed` and role pipes. *Look for:* how `ctx.user` becomes typed non-null.
20. `packages/trpc/server/routers/viewer/eventTypes/util.ts:15-20, 103-199` — resource authz + the PBAC stub. *Look for:* the personal (real) vs. team (stubbed) branches.
21. `packages/features/auth/lib/next-auth-options.ts` (grep `providers`, `jwt`, `session`) — NextAuth config. *Look for:* Credentials/Google/Email providers; the magic-link `maxAge` mismatch.

## Async, UI, and tests (22-25)

22. `packages/features/tasker/task-processor.ts:11-40` — the retry loop. *Look for:* `allSettled`, retry-on-throw, at-least-once.
23. `packages/features/webhooks/lib/sendPayload.ts:296-327` — HMAC signing + `fetch`. *Look for:* what's signed; `redirect:"manual"`.
24. `apps/web/modules/bookings/components/Booker.tsx:1-100` + `packages/features/bookings/Booker/store.ts:442-708` — the client Booker + zustand store. *Look for:* the large prop surface; URL-synced state.
25. `packages/features/bookings/lib/handleNewBooking/test/booking-validations.test.ts` — how tests are shaped here. *Look for:* fixtures, the scenario builders, what layer they test.

## What to ignore (for now)

`packages/embeds/*` internals, `packages/platform/*` atoms, `apps/api/v2` module bodies, `packages/kysely`, the app-store's dozens of integrations, and generated files (`*.generated.ts` — never edit, per `AGENTS.md`). Come back to them from the module that needs them.

Drill: after reading files 6-12, close them and draw the write path from memory. Compare to [05-key-flows.md](05-key-flows.md) Flow 1. Any missing step = re-read that file.
