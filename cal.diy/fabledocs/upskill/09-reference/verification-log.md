# Verification Log

Running log of what was actually inspected while authoring this curriculum. Every claim in these docs traces back to an entry here or to a file re-opened at writing time. Anything not listed was **not** executed or verified.

**Date:** 2026-07-11. **Method:** static reading of the repo (no dev server started, no tests executed, no database created). All commands in the curriculum are marked `verified` only if run here; otherwise `inferred` from `package.json` / `agents/commands.md` / CI workflows.

## Re-check on 2026-10-06 (this study copy)

This is a static re-check against this repository, `elirc/organized-study-cal-com`, snapshot `c08003d`. Nothing was installed, built or run.

- **All five confirmed findings below still hold, at the same lines.** The PBAC stub is at `eventTypes/util.ts:15-20` and is called at 161-167. `delete` is built on it at `_router.ts:104`. The read-then-upsert in `reserveSlot.handler.ts` starts at line 68. The FIXME is at 80-82, and the catch-all `"Event type not found"` is at 104-108. The duplicate `busyTimesModule` load is at `AvailableSlots.ts:36-37`. The `maxAge: 10 * 60 * 60` "valid for 10 min only" comment is at `next-auth-options.ts:362`. The slots cache key `JSON.stringify(args.input)` falls inside 104-120, and `DEFAULT_SLOTS_CACHE_TTL = 2000` is at line 79. The `wc -l` sizes above also still match: 2851, 1435, 1351, 2690, 779 and 679. So do `handler()` at `RegularBookingService.ts:486` and the deps interface at 423.
- **Links:** all 210 relative Markdown links in `fabledocs/upskill/` and `docs/onboarding/` resolve. Of the 916 backticked repo paths, 23 do not exist. Most of them are expected:
  - generated or ignored paths (`packages/prisma/generated/prisma`, `packages/prisma/zod`, `packages/prisma/.env`);
  - files that the spike journals (`docs/onboarding/11`–`13`) say were built and then reverted, such as `apps/web/app/api/health/route.ts` and the `lookupUserByEmail` handlers;
  - a proposed new folder (`packages/features/pbac/`).
- **Fixed in this pass:**
  - the seed path in `00-fast-track.md` now points at `scripts/seed.ts`; the root `package.json` `prisma.seed` entry names a `packages/prisma/seed.ts` that does not exist;
  - `packages/lib/constants` → `constants.ts`;
  - a `task-processor.ts:9-45` anchor in a 42-line file → `9-42`;
  - the `apps/web/modules/teams` and `viewer/teams` paths in story 33 are marked absent;
  - the "if that exact path has no test" hedge in `00-fast-track.md` is removed: `conflictChecker/checkForConflicts.test.ts` and `handleNewBooking/getBookingData.test.ts` both exist.
- **Salvage review (2026-10-07).** The edits above were left uncommitted by an interrupted run and were re-checked line by line before committing: `scripts/seed.ts` exists and `packages/prisma/package.json` maps `seed` → `seed-basic` → `../../scripts/seed.ts`; root `package.json` still names the missing `./packages/prisma/seed.ts`; `task-processor.ts` is 42 lines; `MINUTES_TO_BOOK` is at `constants.ts:137`; `locales/` holds 42 folders with no `en/`; `SECURITY_FILTERED_FILES.md` lists `openapi.json` and both `common.json` files. The duplicate load is in `packages/features/di/containers/AvailableSlots.ts` (the `modules/` file of the same name is a different file). The cache key is at `slots/util.ts:109`.
- **Filtered export.** This copy omits 17 files (repo-root `SECURITY_FILTERED_FILES.md`). Two of the omissions matter to the curriculum. The first is `packages/i18n/locales/en/common.json` (and `fr/`), which the i18n exercises in `00-fast-track.md` and `06-contribution-practice/01-good-first-tickets.md` edit; both places now say so. The second is `docs/api-reference/v2/openapi.json`, so the `docs/api-reference` folder named in `docs/onboarding/01-repo-map.md` is absent. The other filtered files (telemetry, a few tests, platform example READMEs) are not cited.

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
