# Tooling & Build System

## Yarn workspaces

The repo is one big workspace tree (`package.json` `workspaces`): `apps/*`, `apps/api/*`, `packages/*`, `packages/features/*`, `packages/embeds/*`, etc. Packages reference each other by name (`@calcom/lib`, `@calcom/features`, `@calcom/prisma`) and yarn symlinks them. A change in `packages/lib` is instantly visible to `apps/web` without publishing.

Mental model: **`@calcom/*` = a local workspace package.** When you see `import ... from "@calcom/features/..."`, that's a sibling directory, not an npm dependency.

Sharp edge: no barrel imports (`AGENTS.md`) — import the specific file (`@calcom/ui/components/button`) so the build graph stays lean.

Drill: pick three `@calcom/*` imports at the top of `RegularBookingService.ts` and map each to its `packages/` directory.

## Turborepo

`turbo.json` defines tasks (`build`, `type-check`, `lint`, `test`, ...) with dependency awareness and caching. `yarn build` = `turbo run build --filter=@calcom/web...` — the trailing `...` includes upstream deps, so building the web app builds what it needs first. `globalEnv` in `turbo.json` lists env vars that bust the cache (a long list, reflecting how integration-heavy the app is).

Mental model: Turbo is a **task graph with a cache**. If nothing an input touched changed, it replays the cached output. This is why CI is fast and why cache-key correctness (env vars) matters.

Drill: explain why adding a new env var that affects build output must be added to `turbo.json` `globalEnv`.

## Biome (not ESLint/Prettier)

`biome.json` handles both formatting and linting. Notable: it enforces an import-organization order (grouping `@calcom/**`, then `@lib/@components/@server/@trpc`, then `~/`, then relative) — that's why imports in every file look sorted. Run `yarn format` (write) and `yarn lint` / `yarn lint:fix`.

Drill: run `yarn lint` mentally against a file where you added an unsorted import — Biome will reorder it.

## Vitest

`vitest.workspace.ts` selects a suite based on `VITEST_MODE` (default/integration/timezone/packaged-embed) and uses `pool: "forks"` to avoid "closing rpc while fetch was pending" errors. Unit runs force `TZ=UTC` (`package.json` `test`). Timezone tests require `TZ` set or they throw.

Mental model: **one config, several modes.** "Run the tests" means "pick the mode that matches your change" — DB change → integration; timezone math → timezone mode; pure logic → default.

Drill: which mode and command for a change to `checkForConflicts.ts`? (default: `yarn test checkForConflicts`.) For a change to `createBooking.ts` DB write? (`VITEST_MODE=integration`.)

## Playwright

`playwright.config.ts` defines multiple projects (`@calcom/web`, `@calcom/app-store`, `@calcom/embed-core`) with their own `testDir`s and a `webServer` that boots the app. E2E seeds the DB first (`yarn test-e2e` = `db-seed` + `e2e`). Traces land in `apps/web/test-results/`.

Mental model: E2E drives a real browser against a real (seeded) app — slow, high-confidence, reserved for critical flows (booking).

Drill: name the command to run one booking E2E test by name (`yarn e2e <file> --grep "..."`).

## Prisma

Schema-first ORM. `schema.prisma` (2,851 lines) is the source of truth; `yarn db-migrate` creates a migration, `yarn db-deploy` applies, `yarn db-seed` seeds, `yarn db-studio` inspects. Client extensions (`packages/prisma/extensions/`) add cross-cutting behavior (idempotency key). `@zod` schema annotations keep validation aligned.

Drill: describe the safe sequence to add a nullable column to `Booking` (schema edit → `db-migrate` → deploy) and why nullable-first matters for zero-downtime — see [../03-architecture-and-patterns/02-data-model-and-persistence.md](../03-architecture-and-patterns/02-data-model-and-persistence.md).

## Interview angle

- "How do monorepos manage shared code?" → workspaces + Turbo caching.
- "How do you keep CI fast?" → task graph + content-hash cache + correct cache keys.
- "How do you structure tests in a big app?" → modes/layers, seed for E2E, force UTC for determinism. See [08-interview-prep/01](../08-interview-prep/01-js-ts-node-deep-dive.md) Q7 (modules) and [05-quality-engineering/01-testing-strategy.md](../05-quality-engineering/01-testing-strategy.md).
