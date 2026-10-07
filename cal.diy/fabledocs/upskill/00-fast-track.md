# Fast Track — One Weekend in Cal.diy

Goal: by Sunday night you can run the app, trace two end-to-end flows, make one safe change, run one test, and explain a booking's life aloud as if an interviewer asked.

## Saturday morning — run it

All commands `inferred` from root `package.json`, `agents/commands.md`, and `docker-compose.yml` (none were executed while writing this doc — see [09-reference/verification-log.md](09-reference/verification-log.md)).

```bash
# Prereqs: Node 22+ (repo pins yarn@4.12.0 via packageManager in package.json:227), Docker
cp .env.example .env                    # fill NEXTAUTH_SECRET, CALENDSO_ENCRYPTION_KEY, DATABASE_URL
docker compose up -d database redis     # postgres (unicorn_user/magical_password/calendso) + redis — docker-compose.yml:12-33
yarn                                    # install workspace deps
yarn dx                                 # dev setup incl. db (turbo run dx → packages/prisma "dx": "yarn db-setup")
yarn db-seed                            # seed demo users/event types
yarn dev                                # Next.js web app only (turbo --filter=@calcom/web)
```

If `yarn dx` fights you, the manual path is `yarn prisma migrate deploy` (via `yarn db-deploy`) then `yarn db-seed`. Seeded logins are defined in `scripts/seed.ts` (open it and look for the credentials it prints/creates — verify there, don't trust docs). `yarn db-seed` runs `prisma db seed` inside `packages/prisma`, whose `package.json` maps `prisma.seed` to `yarn seed-basic`, which runs `../../scripts/seed.ts`. The root `package.json` also has a `prisma.seed` entry, pointing at `./packages/prisma/seed.ts`, but **that file does not exist in this snapshot**. Don't go looking for it.

Sanity checks: `http://localhost:3000` shows login; a seeded user's public page at `http://localhost:3000/<username>` shows their event types.

## Saturday afternoon — the first 10 files, in order

Open each; the one-liner tells you what to extract. Don't read whole files — read shapes.

1. `README.md` — what Cal.diy is (Cal.com minus enterprise), stack list.
2. root `package.json` — workspaces + scripts. Note `dev` filters to `@calcom/web` (line 47); `test` forces `TZ=UTC` (line 77).
3. `packages/prisma/schema.prisma:851-930` — the `Booking` model. Sit with it: `uid`, `idempotencyKey` (unique, line 855), `status`, `startTime/endTime`, indexes at 918-929. Every feature in this product hangs off this table.
4. `packages/prisma/schema.prisma:156-275` — `EventType`. The product's configuration surface: `length`, `minimumBookingNotice` (default 120, line 219), `seatsPerTimeSlot`, `periodType`, `bookingLimits` (Json), `schedulingType`.
5. `apps/web/pages/api/book/event.ts` (63 lines, read all) — the front door for creating a booking: Turnstile → bot detection → rate limit → session → DI-resolved `RegularBookingService`.
6. `packages/features/bookings/lib/service/RegularBookingService.ts:486-700` — the booking handler's opening acts: load event type, validate reschedule, zod-parse body, blocked-email check, verification, dedupe.
7. `packages/trpc/server/routers/viewer/slots/reserveSlot.handler.ts` (126 lines, read all) — small, complete, and full of judgment calls: seat logic, a reservation race, an embed cookie.
8. `packages/trpc/server/middlewares/sessionMiddleware.ts` — how "logged in" becomes typed `ctx.user` for every authed tRPC procedure.
9. `apps/web/app/(booking-page-wrapper)/[user]/[type]/page.tsx` (73 lines) — App Router page that bridges to legacy `getServerSideProps` via `withAppDirSsr` (lines 15-16). This migration pattern repeats everywhere in `apps/web`.
10. `apps/web/modules/bookings/hooks/useBookings.ts:136-140` — the React Query mutation whose `mutationFn` is `createBooking` → `POST /api/book/event` (`packages/features/bookings/lib/create-booking.ts:5-15`). Now you've seen both ends of the wire.

## Saturday evening — trace flow #1: creating a booking

Follow the money: Booker UI → `useBookings.ts:136` mutation → `create-booking.ts:13` POST → `apps/web/pages/api/book/event.ts:17` handler → `RegularBookingService.ts:486` → `createBooking` (`handleNewBooking/createBooking.ts`) → DB write → emails/webhooks/tasker fan-out.

Pause-and-predict before opening each next file: *what validation must happen before the DB write? who gets notified after?* Compare your list with `RegularBookingService.ts:580-630` (blocked email, spam check, active-bookings limit, email verification). The full trace with a step table is in [01-codebase-cartography/05-key-flows.md](01-codebase-cartography/05-key-flows.md), Flow 1.

## Sunday morning — trace flow #2: where slots come from

`getSchedule.handler.ts:5-8` is deliberately thin — it resolves `AvailableSlotsService` from a DI container (`packages/features/di/containers/AvailableSlots.ts:22-44`, note ~15 modules wired in). The engine is `packages/trpc/server/routers/viewer/slots/util.ts` — skim the imports (lines 1-76) as a table of contents of the availability domain: aggregated availability, conflict checker, booking limits, date ranges, OOO, qualified hosts, Redis cache (`withSlotsCache`, lines 104-120, 2-second TTL at line 79).

Then read `packages/features/bookings/lib/conflictChecker/checkForConflicts.ts` (50 lines) — a clean, self-contained interval-overlap algorithm you could be asked to write on a whiteboard.

## Sunday afternoon — one safe change + one test

1. **Safe change:** in your local tree, pick a UI string on the booking success page and change it (strings live in `packages/i18n/locales/en/common.json` — per `AGENTS.md`, all UI strings go through i18n. **This study export omits `en/common.json` and `fr/common.json`** (see `SECURITY_FILTERED_FILES.md` at the repo root). The other 42 locale folders are present (the `en/` and `fr/` folders are gone entirely), so do this exercise in a full Cal.diy checkout, or read the key's usage in a component and its translation in e.g. `locales/de/common.json`). Watch it hot-reload. Revert.
2. **Run one test file** (`inferred`): `yarn test packages/features/bookings/lib/conflictChecker/checkForConflicts.test.ts` (the file exists in this snapshot; `yarn test checkForConflicts` filters to it too). Also try `yarn test getBookingData` (there's `handleNewBooking/getBookingData.test.ts`). Note the runner: Vitest, `TZ=UTC`, `pool: "forks"` (`vitest.workspace.ts`).

## Sunday evening — teach-back (interview rehearsal)

Explain aloud, 3 minutes, no notes: *"Walk me through what happens when someone books a meeting on a Cal.diy page."* Must include: client mutation → API route defenses (rate limit `event.ts:37-40`, bot check 28-35) → schema-driven body validation (`getBookingData` with event-type-specific `bookingFields`) → availability/conflict re-check → DB write with `idempotencyKey` → side effects (calendar via `EventManager`, emails/SMS, webhooks with HMAC signatures `sendPayload.ts:296-318`, scheduled no-show triggers via tasker).

Self-grade: **Basic** — names the route and the service. **Solid** — includes validation order and one failure mode (double-booking race). **Strong** — explains *why* the slot is re-checked server-side even though the UI only showed free slots, and what `idempotencyKey` protects against.

## What the fast track skips

Recurring bookings, seats, round-robin/team scheduling, payments, the app store, embeds, the NestJS `apps/api/v2`, PBAC, i18n plumbing, and all of testing strategy. The rest of the curriculum covers them; start with [01-codebase-cartography/README.md](01-codebase-cartography/README.md).
