# Runtime & Tooling Map

How the repo is built, tested, and where code runs. Anchors as of 2026-07-11.

## Toolchain

| Tool | Role | Config |
| --- | --- | --- |
| **Yarn 4.12.0** | Package manager, workspaces | `package.json` `packageManager`, `.yarnrc.yml` |
| **Turborepo** | Task runner across the monorepo, caching | `turbo.json` |
| **Next.js** | `apps/web` — App Router + legacy Pages Router | `apps/web/next.config.ts` |
| **NestJS** | `apps/api/v2` — platform REST API | `apps/api/v2/src/app.module.ts` |
| **Prisma** | ORM + schema + migrations | `packages/prisma/schema.prisma` |
| **tRPC** | Typed authenticated API | `packages/trpc/` |
| **Zod** | Runtime validation, type inference | schemas throughout; `@zod` directives in `schema.prisma` |
| **Vitest** | Unit/integration tests | `vitest.workspace.ts`, `vitest.config.mts` |
| **Playwright** | E2E | `playwright.config.ts` |
| **Biome** | Format + lint (not ESLint/Prettier) | `biome.json` |
| **Sentry** | Error/observability | `sentry.*.config.ts`, `withReporting` wrapper |
| **Redis** | Slot cache, rate limiting, redis-tasker backend | `docker-compose.yml`, `packages/features/redis/` |
| **Postgres** | Primary datastore | `docker-compose.yml` (db `calendso`) |

## Runtime boundaries

- **Server (Node):** API routes (`apps/web/pages/api/*`), tRPC handlers, services, repositories, the tasker, webhook signing. Can use `node:*` (e.g. `createHmac`), secrets, and the DB.
- **Server (RSC):** App Router `page.tsx` files run on the server, fetch data (often via the legacy `getServerSideProps` bridge), and pass **serializable** props to client components.
- **Client (browser):** `"use client"` components — the Booker, zustand stores, React Query hooks. No `node:*`, no secrets, no DB. Only `NEXT_PUBLIC_*` env is visible here.
- **Edge:** some Next middleware/config runs at the edge; most logic is Node.
- **Worker:** the tasker processor runs on a schedule/cron (`apps/web/app/api/cron/*`, `.github/workflows/cron-*.yml`).

The serialization boundary bites: `create-booking.ts:8-12` types the response `startTime`/`endTime` as `string` because a `fetch` response can't carry `Date`. Watch for Date↔string at every server→client hop.

## Build & task graph

Turbo runs tasks with dependency awareness (`build` depends on upstream `build`, etc.). `yarn build` = `turbo run build --filter=@calcom/web...` (the `...` includes dependencies). `turbo.json`'s `globalEnv` lists the many env vars that invalidate cache — a hint at how integration-heavy this app is.

## Environment (high level, no secrets)

Configured via `.env` (copy from `.env.example`) and `.env.appStore`. Categories: database URL, `NEXTAUTH_SECRET`/`NEXTAUTH_URL`, encryption keys (`CALCOM_ENCRYPTION_KEY`), Redis, per-integration OAuth client IDs/secrets, feature flags (`NEXT_PUBLIC_*`), Cloudflare Turnstile keys. **Never commit secrets** (`AGENTS.md`). The `dotenv-checker` scripts (`env-check:common`, `env-check:app-store`) validate `.env` against the example schemas.

## Test modes (important nuance)

One `vitest.workspace.ts` switches behavior on `VITEST_MODE`:
- default → unit tests, `TZ=UTC`, `pool: "forks"`.
- `integration` → `*.integration-test.ts` (needs a DB).
- `timezone` → timezone-dependent tests, requires `TZ` set (throws otherwise).
- `packaged-embed` → embed package tests.

So "run the tests" is really "run the right mode." See [../05-quality-engineering/01-testing-strategy.md](../05-quality-engineering/01-testing-strategy.md).

Drill: for a change to timezone-sensitive slot math, which command proves it? (`TZ=<zone> VITEST_MODE=timezone yarn test <file>`.) For a change to the booking DB write? (an `integration` test, since it needs Postgres.)
