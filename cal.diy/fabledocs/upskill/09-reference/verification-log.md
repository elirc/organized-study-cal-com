# Verification Log

Running log of what was actually inspected while authoring this curriculum. Every claim in these docs traces back to an entry here or to a file re-opened at writing time. Anything not listed was **not** executed or verified.

**Date:** 2026-07-11. **Method:** static reading of the repo (no dev server started, no tests executed, no database created). All commands in the curriculum are marked `verified` only if run here; otherwise `inferred` from `package.json` / `agents/commands.md` / CI workflows.

## Environment

- Repo root: `cal.diy/` inside `mastercal/`. Git repo present; branch `main`.
- Windows 11 host. No `.env` present, no `node_modules` inspected for behavior claims.

## Files read (primary evidence)

| Area | Files (line ranges read) | Notes |
| --- | --- | --- |
| Repo identity | `README.md` (1-70), `AGENTS.md` (1-80), `CLAUDE.md`, `agents/commands.md` (1-60) | Cal.diy = MIT community fork of Cal.com, EE removed |
| Workspace | root `package.json` (scripts, workspaces), `turbo.json` (not deep-read), `docker-compose.yml` (1-30), `vitest.workspace.ts` (1-40), `playwright.config.ts` (grep: webServer, projects) | Yarn workspaces + Turborepo; vitest `pool: "forks"`; 3+ Playwright projects |
| Booking API | `apps/web/pages/api/book/event.ts` (entire, 63 lines) | Turnstile, bot detection, rate limit, DI service |
| Booking service | `packages/features/bookings/lib/service/RegularBookingService.ts` (1-140, 380-700 of 2690) | handler() at 486; deps interface at 423-433 |
| Booking submodules | `handleNewBooking/` dir listing; `conflictChecker/checkForConflicts.ts` (entire) | decomposed steps |
| Cancel flow | `packages/features/bookings/lib/handleCancelBooking.ts` (60-179 of ~its length) | guard chain verified |
| Slots | `slots/getSchedule.handler.ts` (entire), `slots/util.ts` (1-120 of 1435), `slots/reserveSlot.handler.ts` (entire, 126 lines) | Redis cache wrapper; TOCTOU seam |
| DI | `packages/features/di/containers/AvailableSlots.ts` (entire, 45 lines) | duplicate `busyTimesModule` load at 36-37 |
| tRPC infra | `packages/trpc/server/trpc.ts` (1-50), `middlewares/sessionMiddleware.ts` (1-60), procedures dir listing | superjson + errorFormatter |
| EventTypes authz | `packages/trpc/server/routers/viewer/eventTypes/util.ts` (1-199), `_router.ts` (grep lines 8, 21, 104-109, 156-226) | **PBAC stub always returns true — confirmed at 15-20, called at 161-167** |
| Auth | `packages/features/auth/lib/` dir listing; `next-auth-options.ts` (grep: providers, jwt, session, maxAge) | magic-link maxAge comment/value mismatch at ~359-362 |
| Schema | `packages/prisma/schema.prisma` (156-275, 851-970, 1437-1465, 1825-1860; model-line grep for the rest) | 2851 lines total |
| Tasker | `packages/features/tasker/tasker.ts` (entire), `task-processor.ts` (1-80), `tasks/` dir listing, `Task` model | internal vs redis tasker |
| Webhooks | `packages/features/webhooks/lib/` dir listing; `sendPayload.ts` (280-328; grep for HMAC) | HMAC SHA-256, `redirect: "manual"` |
| Booker UI | `Booker/store.ts` (1-70; grep 442, 708), `apps/web/modules/bookings/components/Booker.tsx` (1-80), `hooks/useBookings.ts` (grep 12, 136-137, 418-429), `lib/create-booking.ts` (entire) | zustand store; React Query mutation → POST /api/book/event |
| Booker page (SSR) | `apps/web/app/(booking-page-wrapper)/[user]/[type]/page.tsx` (entire, 73 lines) | App Router → legacy `getServerSideProps` bridge |
| EventManager | `EventManager.ts` (grep method lines: 133, 287, 615, 853, 873, 1079) | not deep-read; method existence only |
| App store | `packages/app-store/googlecalendar/lib/CalendarService.ts` (grep: 179, 364, 463, 540, 703) | method existence only |
| Rate limiting | `packages/lib/checkRateLimitAndThrowError.ts` (1-40) | 429 with reset seconds |
| API v2 | `apps/api/v2/src` top-level + `modules/` dir listing | NestJS; not deep-read |
| CI | `.github/workflows/` dir listing (~20 of the files) | names only; YAML not deep-read |

## Commands run

| Command | Result |
| --- | --- |
| `ls` / `rg` / `grep -n` file inspection commands | as logged above |
| `wc -l` on key files | `schema.prisma` 2851; `slots/util.ts` 1435; `EventManager.ts` 1351; `RegularBookingService.ts` 2690; `Booker/store.ts` 779; `scheduleTrigger.ts` 679 |
| No build/test/dev commands executed | all runtime commands in the docs are `inferred` |

## Confirmed findings (high confidence)

1. **PBAC permission stub** — `eventTypes/util.ts:15-20` defines a local `PermissionCheckService` whose `checkPermission` returns `true` unconditionally; `createEventPbacProcedure` calls it for team events at `util.ts:161-167`. Team-event membership is *not* checked in that branch (only `input.users` assignment validation at 178-196). The `delete` procedure is built on this (`_router.ts:104-109`).
2. **Slot reservation TOCTOU** — `reserveSlot.handler.ts:68-103`: read (`findReservedByOthers`) then write (`upsert`) without a transaction; two concurrent requests can both pass the check. Also a `FIXME` at 80-82 (team events reserve against the wrong users) and a swallowed catch that rethrows misleading "Event type not found" at 104-109.
3. **Duplicate DI module load** — `di/containers/AvailableSlots.ts:36-37` loads `busyTimesModule` twice under the same token (harmless; reviewable nit).
4. **Magic-link maxAge mismatch** — `next-auth-options.ts` (~line 362): `maxAge: 10 * 60 * 60` (10 hours) with comment "valid for 10 min only". One of the two is wrong.
5. **Slots Redis cache key** — `slots/util.ts:104-120` keys the cache on `JSON.stringify(args.input)`; TTL constant `DEFAULT_SLOTS_CACHE_TTL = 2000` (ms) at line 79. Key stability depends on property order of the input object.

## Uncertainties / not covered

- Did not run the app; all `yarn` command behavior inferred from manifests.
- `packages/features` has 80+ subsystems; only those listed above were read. Payments (Stripe app), CRM, credits, SMS, embeds internals, `apps/api/v2` module code, and `packages/kysely` usage were surveyed by directory listing only.
- Line anchors for `EventManager.ts` and `googlecalendar/CalendarService.ts` are method-start lines from grep, not full-body reads.
- Whether any *other* layer re-checks team permissions above finding #1 (e.g., in individual handlers) was spot-checked for `delete.handler` route wiring only — treat downstream handler compensation as **unverified**.
- git history: repo has commits (branch `main`), but history/`git log` output was not inspected for authorship claims.
