# Repo Map

This repository is a Yarn/Turborepo monorepo. The mental model is:

- `apps/*` contains runnable products.
- `packages/*` contains shared libraries, domain features, generated clients, UI, app integrations, and infrastructure code.
- Root config files coordinate builds, tests, formatting, package management, and environment rules.

## Top-level files

### `package.json`

The root manifest defines workspaces, shared scripts, and tool versions. Start here when you need to know how the project is installed, built, tested, or run locally.

Important things in this file:

- `packageManager: "yarn@4.12.0"` means the repo expects Yarn Berry.
- `workspaces` includes `apps/*`, `packages/*`, `packages/features/*`, `packages/app-store/*`, `packages/platform/*`, and `example-apps/*`.
- Scripts such as `dev`, `dx`, `build`, `test`, `test-e2e`, `type-check`, `lint`, `db-deploy`, and `db-studio` are orchestration commands, usually delegated through Turborepo.
- Prisma configuration points at `packages/prisma/schema.prisma`; seed helpers are exposed through package scripts such as `seed-basic`, `seed-app-store`, and `seed-pbac`.

Touch it when adding a workspace, changing root scripts, upgrading workspace-wide tooling, or adjusting Prisma seed/schema config.

### `turbo.json`

Defines the build graph. Turborepo uses it to understand task dependencies, cache behavior, persistent dev tasks, and environment variables that affect outputs.

Touch it when adding or changing cross-package scripts, changing cache inputs/outputs, or introducing a new env var that must invalidate builds.

### `.env.example`

The reference environment file. It documents required local variables like `DATABASE_URL`, `NEXTAUTH_SECRET`, `CALENDSO_ENCRYPTION_KEY`, `NEXT_PUBLIC_WEBAPP_URL`, and integration credentials.

Touch it whenever a new environment variable is required for local development or production.

### `docker-compose.yml`

Defines local infrastructure and containerized services such as the database, Redis, app, API, and Prisma Studio.

Touch it when changing local service dependencies or Docker-based development workflows.

### `vitest.workspace.ts` and `vitest.config.mts`

Vitest project configuration. The workspace splits test behavior across packages and specialized modes, including booking integration tests, embed tests, app-store tests, and UI tests.

Touch these when changing test grouping, aliases, setup files, or adding a new package with special test needs.

### `playwright.config.ts`

End-to-end test configuration. It starts the web app, configures browsers, and writes reports under `test-results`.

Touch it when changing E2E server startup, browser projects, retries, or reporting.

### `agents/rules`

Repo-specific engineering rules. These are not runtime code, but they explain team conventions around repositories, services, DI, Prisma usage, error handling, and PR shape.

Read it before adding new service/repository code.

## `apps/`

Runnable applications live here.

### `apps/web/`

The main Cal.diy web application. This is the scheduling UI most developers will spend time in.

What lives here:

- `app/`: Next.js App Router routes, layouts, wrappers, and app-level providers.
- `pages/`: legacy Next.js Pages Router routes and API routes.
- `modules/`: web-facing feature UI modules such as auth, bookings, event types, teams, settings, and workflows.
- `components/`: shared web-only components.
- `lib/`: web app helpers, server helpers, routing helpers, and compatibility glue.
- `server/`: server-side loaders used by route wrappers, including legacy-style `getServerSideProps` helpers used from App Router pages.
- `styles/`: app-level styling.

Touch it when changing screens, route behavior, web API routes, app shell behavior, or frontend interaction flows.

Key files to know:

- `apps/web/app/(booking-page-wrapper)/[user]/[type]/page.tsx`: public booking page App Router entry.
- `apps/web/server/lib/[user]/[type]/getServerSideProps.ts`: server loader for public event pages.
- `apps/web/modules/bookings/components/Booker.tsx`: core booking UI state machine.
- `apps/web/pages/api/book/event.ts`: web API endpoint that submits a booking request.
- `apps/web/pages/api/auth/[...nextauth].ts`: NextAuth API endpoint.
- `apps/web/app/_trpc/trpc-provider.tsx`: React provider for tRPC and TanStack Query.
- `apps/web/app/_trpc/trpc-client.ts`: browser tRPC client setup.

Non-obvious: this app uses both App Router and Pages Router. New route UI may live under `app/`, while many API routes and legacy pages still live under `pages/`.

### `apps/api/`

The public API surface. In this repo the important app is `apps/api/v2/`, a NestJS service.

What lives here:

- Nest modules, controllers, services, guards, DTOs, jobs, Swagger/OpenAPI config, and tests.
- API-specific build and test scripts.

Touch it when changing API v2 behavior, public API contracts, Swagger docs, background jobs, API auth, or platform-facing endpoints.

### `apps/docs/`

The documentation site. It uses Next/Nextra/Pagefind and is separate from the onboarding docs you are reading.

Touch it when changing public documentation pages, docs navigation, search, or docs-site behavior.

## `packages/`

Shared libraries and domain packages live here.

### `packages/features/`

The domain-feature layer. This is where much of the product behavior lives.

What lives here:

- `bookings/`: booking creation, rescheduling, validation, payment, notifications, calendar event creation, and booking-related UI/domain helpers.
- `eventtypes/`: event type forms, schemas, repositories, creation/update logic, and event type UI.
- `auth/`: NextAuth options, credential authorization, session helpers, two-factor auth, and auth-related repositories.
- `availability/`, `schedules/`, `slots/`: schedule and slot computation helpers.
- `webhooks/`, `membership/`, `ooo/`, `selectedCalendar/`, `slots/`, `users/`, and other feature folders.

Touch it when changing business behavior. A good rule: if the code answers "what does this scheduling feature mean?", it usually belongs here.

Non-obvious: feature packages are meant to expose public APIs across boundaries. Avoid reaching deep into another feature's internals when a public function, repository, or service exists.

### `packages/trpc/`

The typed RPC API layer used by the web app.

What lives here:

- `server/routers/`: tRPC routers and handlers.
- `server/procedures/`: `publicProcedure`, `authedProcedure`, and other procedure wrappers.
- `server/middlewares/`: auth/session/error middleware.
- `server/createContext.ts`: request context, including Prisma clients and session.
- `react/`: shared client-side tRPC helpers and endpoint names.

Touch it when adding or changing web app API procedures used through `trpc.viewer.*`.

Key files:

- `packages/trpc/server/routers/_app.ts`
- `packages/trpc/server/routers/viewer/_router.tsx`
- `packages/trpc/server/trpc.ts`
- `packages/trpc/server/createNextApiHandler.ts`
- `packages/trpc/server/createContext.ts`

Non-obvious: tRPC procedures are split into endpoint groups, and the browser client maps router paths to endpoint names using `packages/trpc/react/shared.ts`.

### `packages/prisma/`

Database schema, generated Prisma client, migrations, seeds, and Prisma utility exports.

What lives here:

- `schema.prisma`: source of truth for database models.
- `migrations/`: database migrations.
- seed helpers such as `seed-pbac-only.ts`; broader seed scripts are wired through `packages/prisma/package.json`.
- `index.ts`: exports `prisma`, `readonlyPrisma`, and `customPrisma` with extensions and adapters.
- generated output under `generated/` and `zod/`.

Touch it when changing data models, indexes, relations, seeds, migrations, or Prisma client behavior.

### `packages/lib/`

Shared low-level utilities used throughout the repo.

What lives here:

- Constants, environment helpers, fetch wrappers, logging, telemetry, common validation helpers, date/time utilities, permissions helpers, and generic service utilities.

Touch it when the code is broadly reusable and not tied to one feature. Be cautious: changes here can have wide blast radius.

### `packages/ui/`

Shared design system and React UI primitives.

What lives here:

- Buttons, dialogs, forms, inputs, layout components, icons/wrappers, styling primitives, and reusable UI helpers.

Touch it when a UI component is genuinely shared across the product.

### `packages/coss-ui/`

Additional UI package used by the project. Treat it as a shared UI workspace and check existing imports before adding new UI primitives elsewhere.

### `packages/app-store/`

Integration app definitions and provider-specific implementations.

What lives here:

- Calendar, conferencing, CRM, payment, webhook, analytics, and communication integrations.
- Per-app package manifests for many integrations.
- App metadata, credential handling, API adapters, and generated/static assets.

Touch it when changing a third-party integration such as Google Calendar, Zoom, Stripe, HubSpot, or Cal Video provider behavior.

### `packages/app-store-cli/`

Tooling for managing app-store metadata and assets.

Touch it when changing how app-store packages are generated or maintained.

### `packages/platform/`

Platform/API client pieces, atoms, libraries, examples, constants, enums, and types.

What lives here:

- `atoms/`: reusable platform-facing hooks and components, including booking hooks used by the web booker.
- `libraries/`, `types/`, `constants/`, `enums/`, `utils/`: platform package support.
- `examples/`: example platform apps.

Touch it when changing platform SDK/API behavior or shared hooks consumed by embeddable/platform flows.

### `packages/embeds/`

Embed packages for using Cal.diy booking experiences inside other sites.

What lives here:

- `embed-core`, `embed-react`, `embed-snippet`, and related embed packages.

Touch it when changing embeddable booking behavior, iframe communication, embed SDK events, or React embed exports.

### `packages/emails/`

Email rendering and templates.

Touch it when changing booking confirmations, cancellation emails, reminders, workflow emails, or other transactional mail.

### `packages/i18n/`

Translations and internationalization support.

Touch it when adding translation keys, locales, or changing how text is loaded.

### `packages/dayjs/`

Shared Day.js configuration and date helpers.

Touch it when changing global date/time behavior. Calendar software is date-heavy, so treat this package carefully.

### `packages/kysely/`

Kysely-related generated database typing and query support.

Touch it when working on query code that uses Kysely instead of Prisma, or when updating generated database types.

### `packages/config/`, `packages/tsconfig/`, `packages/types/`

Shared configuration and type packages.

Touch these when changing TypeScript baselines, shared config exports, or cross-package types.

### `packages/testing/`

Shared test helpers.

Touch it when adding reusable fixtures, mocks, factories, or test utilities.

### `packages/debugging/` and `packages/sms/`

Narrow shared packages for debugging helpers and SMS behavior.

Touch them only when working in those specific areas.

## `example-apps/`

Example consumers of the platform/API packages. Use these to understand external usage patterns and to manually validate SDK/platform behavior.

## `docs/`

Documentation. Existing docs include API reference material under `docs/api-reference`. The onboarding docs live in `docs/onboarding`.

Touch this directory when creating internal developer documentation.

## `tests/`, `test-results/`, and generated output

E2E tests and outputs are scattered by package and config. Generated and result directories should not be treated as source of truth.

When looking for tests, use:

```powershell
rg --files | rg "\.(test|spec)\.(ts|tsx)$"
```

## Where to go for common tasks

Change the public booking page:

- Start in `apps/web/app/(booking-page-wrapper)/[user]/[type]/page.tsx`.
- Follow into `apps/web/server/lib/[user]/[type]/getServerSideProps.ts`.
- UI behavior usually lands in `apps/web/modules/bookings/components/Booker.tsx`.

Change booking creation:

- Start in `packages/features/bookings/lib/service/RegularBookingService.ts`.
- Check `apps/web/pages/api/book/event.ts` for the web entry point.
- Check `packages/features/bookings/lib/handleNewBooking/createBooking.ts` for persistence.

Add or change a tRPC endpoint:

- Register it under `packages/trpc/server/routers/viewer/_router.tsx`.
- Add the specific route/handler under the relevant `packages/trpc/server/routers/viewer/*` folder.
- Use the frontend through `trpc.viewer.<area>.<procedure>`.

Change event type creation:

- UI: `apps/web/modules/event-types/components/CreateEventTypeDialog.tsx`.
- Hook: `apps/web/modules/event-types/hooks/useCreateEventType.ts`.
- API handler: `packages/trpc/server/routers/viewer/eventTypes/heavy/create.handler.ts`.
- Persistence: `packages/features/eventtypes/repositories/eventTypeRepository.ts`.

Change authentication:

- UI: `apps/web/modules/auth/login-view.tsx`.
- API: `apps/web/pages/api/auth/[...nextauth].ts`.
- Auth options: `packages/features/auth/lib/next-auth-options.ts`.
- Session lookup: `packages/features/auth/lib/userFromSessionUtils.ts`.

Change database schema:

- Edit `packages/prisma/schema.prisma`.
- Add a migration.
- Update code that reads/writes the changed model.
- Update `.env.example` only if new database-related env is required.
