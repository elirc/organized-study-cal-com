# Tech Stack Explainer

This project uses a modern TypeScript monorepo stack with Next.js for the main web app, NestJS for API v2, tRPC for typed web APIs, Prisma for database access, and Turborepo/Yarn for workspace orchestration.

## Runtime and package management

### Node.js

The README requires Node `>=18.x`. All apps and packages run on Node-based tooling.

Where it shows up:

- Root `package.json`
- App/package scripts
- Docker setup
- Next.js, NestJS, Prisma, Vitest, and Playwright commands

### Yarn 4

The repo uses Yarn Berry, pinned as `yarn@4.12.0` in the root `package.json`.

Where it shows up:

- Root `package.json`
- `.yarnrc.yml`
- Workspace scripts such as `yarn dev`, `yarn dx`, `yarn test`, and `yarn workspace @calcom/web dev`

Project-specific note: this repo uses a monorepo workspace layout, so packages import each other by workspace package names such as `@calcom/prisma`, `@calcom/trpc`, `@calcom/features`, and `@calcom/lib`.

### Turborepo

Turborepo coordinates tasks across many packages. It figures out what needs to run, what can be cached, and which tasks depend on each other.

Where it shows up:

- `turbo.json`
- Root scripts such as `dev`, `build`, `test`, `type-check`, and `lint`

In this project:

- `@calcom/web#dev` depends on environment checks and app-store static copy tasks.
- `@calcom/api-v2#build` is defined as a separate build target.
- Global env vars in `turbo.json` are part of cache correctness.

## Main app framework

### Next.js

`apps/web` is the main web app. It uses Next.js with both App Router and Pages Router.

Where it shows up:

- `apps/web/app/`: App Router routes and layouts.
- `apps/web/pages/`: legacy pages and API routes.
- `apps/web/next.config.ts` and related Next config files.
- `apps/web/package.json`

In this project:

- Public booking pages enter through App Router, for example `apps/web/app/(booking-page-wrapper)/[user]/[type]/page.tsx`.
- Many API routes still use Pages Router, for example `apps/web/pages/api/book/event.ts` and `apps/web/pages/api/auth/[...nextauth].ts`.
- Some App Router pages call legacy-style server loaders under `apps/web/server/lib/*/getServerSideProps.ts`.

### React

React powers the web UI. Components are split between app-specific modules in `apps/web/modules`, shared UI in `packages/ui`, and feature-specific components in `packages/features`.

Where it shows up:

- `apps/web/modules/**`
- `apps/web/components/**`
- `packages/ui/**`
- `packages/features/*/components/**`
- `packages/platform/atoms/**`

In this project:

- Booking UI is centered on `apps/web/modules/bookings/components/Booker.tsx`.
- Event type creation uses `apps/web/modules/event-types/components/CreateEventTypeDialog.tsx` and `packages/features/eventtypes/components/CreateEventTypeForm.tsx`.

### TypeScript

The whole repo is TypeScript-first.

Where it shows up:

- Nearly all app/package source files.
- Shared config in `packages/tsconfig`.
- Type checks through `yarn type-check`.

Project-specific note: path aliases and workspace packages are used heavily. When tracing imports, check both local relative files and package imports like `@calcom/features/bookings/...`.

## API layers

### tRPC

tRPC provides type-safe RPC calls between the web frontend and server code. It is the main internal API layer for authenticated web app operations.

Where it shows up:

- Client:
  - `apps/web/app/_trpc/trpc.ts`
  - `apps/web/app/_trpc/trpc-client.ts`
  - `apps/web/app/_trpc/trpc-provider.tsx`
- Server:
  - `packages/trpc/server/routers/_app.ts`
  - `packages/trpc/server/routers/viewer/_router.tsx`
  - `packages/trpc/server/createContext.ts`
  - `packages/trpc/server/procedures/publicProcedure.ts`
  - `packages/trpc/server/procedures/authedProcedure.ts`

In this project:

- Most product procedures live under `trpc.viewer.*`.
- Routers are grouped by domain, such as `viewer.eventTypesHeavy.create`, `viewer.slots.getSchedule`, and `viewer.bookings`.
- `publicProcedure` and `authedProcedure` wrap common middleware.
- `errorConversionMiddleware` converts project errors into `TRPCError`.
- The frontend client maps procedure paths to endpoint groups using `packages/trpc/react/shared.ts`.

### Next.js API routes

Some endpoints are normal Next API routes rather than tRPC.

Where it shows up:

- `apps/web/pages/api/book/event.ts`
- `apps/web/pages/api/auth/[...nextauth].ts`
- `apps/web/pages/api/trpc/**`

In this project:

- Booking creation from the public booker posts to `/api/book/event`, not directly to tRPC.
- NextAuth uses the standard catch-all route under `pages/api/auth`.

### NestJS

API v2 is a NestJS app under `apps/api/v2`.

Where it shows up:

- `apps/api/v2/package.json`
- Nest modules, controllers, services, guards, and tests under `apps/api/v2`

In this project:

- API v2 is the public/platform API surface.
- It uses Nest concepts such as modules, controllers, services, guards, Passport/JWT auth, Swagger/OpenAPI, Bull queues, Redis, Prisma, and Jest.

## Database and persistence

### PostgreSQL

Postgres is the primary database.

Where it shows up:

- `DATABASE_URL` and `DATABASE_DIRECT_URL` in `.env.example`
- `docker-compose.yml`
- Prisma schema and migrations under `packages/prisma`

### Prisma

Prisma is the main ORM and schema source of truth.

Where it shows up:

- `packages/prisma/schema.prisma`
- `packages/prisma/index.ts`
- `packages/prisma/migrations/`
- `packages/features/*/repositories/**`
- Service code that reads/writes domain data

In this project:

- Prisma client output is generated under `packages/prisma/generated/prisma`.
- The schema also generates Zod types and Kysely types.
- `packages/prisma/index.ts` exports `prisma`, `readonlyPrisma`, and `customPrisma`.
- The project prefers repository classes for data access, but older code still uses Prisma directly in some services.

### Kysely

Kysely is a typed SQL query builder used in parts of the codebase where direct SQL-style queries are preferred.

Where it shows up:

- `packages/kysely`
- Prisma generator config in `packages/prisma/schema.prisma`

In this project:

- Prisma remains the main ORM.
- Kysely support exists for typed query paths and generated DB types.

## Auth and security

### NextAuth

NextAuth handles web authentication.

Where it shows up:

- `apps/web/pages/api/auth/[...nextauth].ts`
- `packages/features/auth/lib/next-auth-options.ts`
- `packages/features/auth/lib/userFromSessionUtils.ts`

In this project:

- Credentials auth is custom and checks password, lock state, rate limits, two-factor auth, backup codes, and inactive-admin rules.
- Providers can include credentials, email, Google, and Azure AD depending on environment variables.
- Sessions use JWT strategy and are enriched by reloading the user from the database.

### Zod

Zod validates API input and shared schemas.

Where it shows up:

- tRPC route schemas, for example `packages/trpc/server/routers/viewer/eventTypes/heavy/create.schema.ts`
- Feature schemas such as `packages/features/eventtypes/lib/schemas.ts`
- Booking body schemas such as `packages/features/bookings/lib/bookingCreateBodySchema.ts`

In this project:

- tRPC procedures usually declare `.input(SomeZodSchema)`.
- Booking creation validates the web API body before service work.

### Sentry

Sentry is used for error reporting and tracing.

Where it shows up:

- Web and API dependencies
- Error/session middleware
- Build scripts such as `sentry:release`

In this project:

- tRPC auth middleware sets Sentry user context when a user is authenticated.
- Booking and API paths pass trace context through service layers.

## Frontend state and data fetching

### TanStack Query

TanStack Query powers client-side server state caching. tRPC uses it under the hood on the client.

Where it shows up:

- `apps/web/app/_trpc/trpc-provider.tsx`
- Hooks using `trpc.viewer.*.useQuery()` or `.useMutation()`
- Direct mutations such as booking creation in `apps/web/modules/bookings/hooks/useBookings.ts`

### React Hook Form

React Hook Form powers form state.

Where it shows up:

- `apps/web/modules/bookings/components/BookEventForm/BookEventForm.tsx`
- `packages/features/eventtypes/components/CreateEventTypeForm.tsx`
- Many settings/forms modules

In this project:

- Forms often combine React Hook Form, Zod schemas, and UI components from `packages/ui`.

### Zustand and local stores

The booking flow uses a local store/state-machine style pattern around the booker.

Where it shows up:

- `apps/web/modules/bookings/components/Booker.tsx`
- Booker store imports used by booking components and platform atoms

In this project:

- The booker moves through states like `loading`, `selecting_date`, `selecting_time`, and `booking`.

### Jotai

Jotai is available and used for smaller atom-based state in parts of the app.

Where it shows up:

- Search imports with `rg "from \"jotai\"|from 'jotai'"`.

## Styling and UI

### Tailwind CSS

Tailwind provides utility-first styling.

Where it shows up:

- Web app styles
- UI package components
- Class names across React components

### Radix UI

Radix provides accessible low-level UI primitives.

Where it shows up:

- `packages/ui`
- Dialogs, popovers, menus, tooltips, and similar primitives

### Lucide React

Lucide provides icons.

Where it shows up:

- UI components and app screens importing icons from `lucide-react`.

### Sonner

Sonner is used for toast notifications.

Where it shows up:

- Event type creation and many mutation flows.

## Calendar, scheduling, and integrations

### Day.js

Day.js handles date/time utilities.

Where it shows up:

- `packages/dayjs`
- Booking, schedule, timezone, and slot logic throughout features

In this project:

- Timezone correctness is critical. Look for project helpers before hand-writing date math.

### App Store integrations

Calendar, conferencing, payment, CRM, and messaging integrations live under `packages/app-store`.

Where it shows up:

- Calendar availability and event creation
- Booking references
- Credentials
- Conferencing links
- Payment flows

### Daily / Cal Video

Cal Video appears as the built-in video/conferencing option.

Where it shows up:

- Event type Cal Video settings
- Booking references
- App-store/provider code

## Testing

### Vitest

Vitest is the primary unit/integration test runner.

Where it shows up:

- `vitest.workspace.ts`
- `vitest.config.mts`
- `*.test.ts`, `*.test.tsx`, `*.spec.ts`, `*.spec.tsx`

Use it for package tests and most feature/service tests.

### Playwright

Playwright runs browser E2E tests.

Where it shows up:

- `playwright.config.ts`
- E2E tests and reports under `test-results`

Use it for full browser flows, especially booking, auth, and major UI flows.

### Jest

API v2 uses Jest through Nest tooling.

Where it shows up:

- `apps/api/v2/package.json`
- API v2 tests

## Documentation

### Nextra and Pagefind

The docs app uses Nextra and Pagefind for docs rendering and search.

Where it shows up:

- `apps/docs/package.json`

Touch this when working on the docs site, not when changing the app itself.
