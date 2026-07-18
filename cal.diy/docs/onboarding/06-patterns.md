# Patterns and Conventions

This codebase is large, but it has recognizable patterns. A good PR from this team usually keeps transport code thin, puts domain behavior in feature packages, validates inputs with schemas, keeps database access in repositories where practical, and updates tests/docs close to the behavior changed.

## Monorepo conventions

### Apps vs packages

- Runnable products live in `apps/*`.
- Shared/domain code lives in `packages/*`.
- Feature-specific domain code lives in `packages/features/*`.

Common pattern:

```text
apps/web module
  -> packages/trpc route or Next API route
  -> packages/features service/handler
  -> repository or Prisma
  -> packages/prisma/schema.prisma
```

### Workspace imports

Imports commonly use workspace aliases:

- `@calcom/prisma`
- `@calcom/trpc`
- `@calcom/features`
- `@calcom/lib`
- `@calcom/ui`
- `@calcom/platform-*`

When tracing behavior, do not assume an import is third-party just because it starts with `@calcom`.

## File and folder naming

Rules from repo guidance and observed code:

- Repository classes use PascalCase with a `Repository` suffix, for example `PrismaAppRepository.ts` and `eventTypeRepository.ts` with `EventTypeRepository`.
- Service classes use names like `<Entity>Service.ts`, not `.service.ts` for new files.
- Tests use `.test.ts`, `.test.tsx`, `.spec.ts`, or `.spec.tsx`.
- tRPC domain folders usually group `_router.ts`, `*.schema.ts`, `*.handler.ts`, and tests.

Examples:

- `packages/trpc/server/routers/viewer/eventTypes/heavy/_router.ts`
- `packages/trpc/server/routers/viewer/eventTypes/heavy/create.schema.ts`
- `packages/trpc/server/routers/viewer/eventTypes/heavy/create.handler.ts`
- `packages/trpc/server/routers/viewer/slots/getSchedule.handler.ts`
- `packages/trpc/server/routers/viewer/slots/getSchedule.schema.ts`

## Frontend organization

### `apps/web/app`

Next.js App Router routes, layouts, and providers.

Use it for route entry points and app-level structure.

Example:

- `apps/web/app/(booking-page-wrapper)/[user]/[type]/page.tsx`

### `apps/web/pages`

Legacy Pages Router routes and API routes.

Important examples:

- `apps/web/pages/api/book/event.ts`
- `apps/web/pages/api/auth/[...nextauth].ts`
- `apps/web/pages/api/trpc/**`

Pattern to remember:

- New UI can enter from App Router while important API routes still live in Pages Router.

### `apps/web/modules`

Feature-oriented UI modules for the web app.

Examples:

- `apps/web/modules/bookings`
- `apps/web/modules/event-types`
- `apps/web/modules/auth`
- `apps/web/modules/users`

Use it for app-specific screens, hooks, and UI composition.

### `packages/features/*/components`

Shared feature components that are not only web-app-specific.

Example:

- `packages/features/eventtypes/components/CreateEventTypeForm.tsx`

Use it when the component belongs to a domain feature and is reused across app/platform surfaces.

### `packages/ui`

Shared UI primitives and design-system components.

Use it for reusable components, not one-off feature screens.

## Component patterns

### Container/coordinator components

Large components such as `apps/web/modules/bookings/components/Booker.tsx` coordinate state and delegate work.

Expected pattern:

- UI state in hooks/stores.
- Form state in React Hook Form.
- Server state in tRPC/TanStack Query.
- Business rules in feature packages.

### Form components

Forms often use:

- React Hook Form for form state.
- Zod for validation schemas.
- `packages/ui` form controls.
- Feature-specific schema helpers.

Examples:

- `apps/web/modules/bookings/components/BookEventForm/BookEventForm.tsx`
- `packages/features/eventtypes/components/CreateEventTypeForm.tsx`

## State management

### Server state

tRPC hooks and TanStack Query are the default for fetched server state.

Example:

- `trpc.viewer.eventTypesHeavy.create.useMutation()` in `apps/web/modules/event-types/hooks/useCreateEventType.ts`.

### Local UI state

React state, hooks, Zustand-like stores, and feature stores are used for local interaction state.

Example:

- The Booker uses state values such as `loading`, `selecting_date`, `selecting_time`, and `booking`.

### Forms

React Hook Form is the common choice for user input.

### Small atom state

Jotai is available and used in parts of the app for atom-style state.

## API route patterns

### tRPC routers

Router shape:

```ts
export const someRouter = router({
  create: authedProcedure.input(SomeSchema).mutation(async ({ ctx, input }) => {
    return createHandler({ ctx, input });
  }),
});
```

Common pieces:

- `_router.ts`: registers procedures.
- `*.schema.ts`: Zod input schemas.
- `*.handler.ts`: procedure implementation.
- `*.test.ts`: tests.

Root registration:

- `packages/trpc/server/routers/_app.ts`
- `packages/trpc/server/routers/viewer/_router.tsx`

Procedure wrappers:

- `packages/trpc/server/procedures/publicProcedure.ts`
- `packages/trpc/server/procedures/authedProcedure.ts`

Context:

- `packages/trpc/server/createContext.ts`

### Next API routes

Next API routes are used for special web endpoints.

Example:

- `apps/web/pages/api/book/event.ts`

Expected pattern:

- Parse/validate request.
- Run transport/security checks.
- Delegate to feature service.
- Wrap with responder helpers such as `defaultResponder`.

### NestJS API v2

API v2 follows Nest patterns:

- controllers for HTTP surface
- services for business logic
- modules for dependency grouping
- guards for auth
- DTOs/schemas for input/output
- Swagger/OpenAPI for docs

Look under `apps/api/v2`.

## Service and repository patterns

### Services

Services orchestrate domain workflows.

Example:

- `packages/features/bookings/lib/service/RegularBookingService.ts`

Use services for:

- multi-step behavior
- validation that spans models
- side effects
- integration orchestration

### Repositories

Repositories isolate database access where possible.

Example:

- `packages/features/eventtypes/repositories/eventTypeRepository.ts`

Repo guidance prefers:

- repository classes for Prisma access
- explicit methods named by domain intent
- Prisma `select` over broad `include` when possible

Reality check:

- Older code and large services sometimes call Prisma directly. When adding new code, prefer the repository pattern unless the surrounding code clearly uses another local convention.

## Dependency injection

Some feature services use `@evyweb/ioctopus` DI.

Examples:

- `packages/features/bookings/di/RegularBookingService.container.ts`
- `packages/features/bookings/di/RegularBookingService.module.ts`

Pattern:

- Define tokens.
- Bind modules/classes.
- Resolve through container helpers.

Use this pattern when extending a service that already uses DI. Do not introduce a new DI style.

## Error handling

### tRPC errors

tRPC handlers should throw `TRPCError` or project errors that middleware converts.

Important files:

- `packages/trpc/server/middlewares/errorConversionMiddleware.ts`
- `packages/trpc/server/trpc.ts`

### Project errors

Service/repository code may use project-specific errors such as `ErrorWithCode`.

Pattern:

- Domain/service code throws meaningful project errors.
- tRPC middleware converts them for the API response.
- API routes use responder helpers and route-level error handling.

### User-facing errors

Frontend mutation hooks commonly map server errors into toasts or form errors.

Example:

- `apps/web/modules/event-types/hooks/useCreateEventType.ts` maps duplicate slug and unauthorized errors.

## Environment variables and config

### Source of truth

Use `.env.example` to document required variables.

Important local variables:

- `DATABASE_URL`
- `DATABASE_DIRECT_URL`
- `NEXT_PUBLIC_WEBAPP_URL`
- `NEXT_PUBLIC_WEBSITE_URL`
- `NEXTAUTH_URL`
- `NEXTAUTH_SECRET`
- `CALENDSO_ENCRYPTION_KEY`
- integration-specific client IDs/secrets

### Turborepo cache awareness

If a variable affects builds or task outputs, check `turbo.json`. Global env configuration there affects caching.

### Runtime constants

Common runtime constants live in `packages/lib` and app config files.

Pattern:

- Do not scatter `process.env.X` checks across unrelated files if a project helper already exists.
- Add new variables to `.env.example`.
- Add validation/documentation where the project expects it.

## Validation conventions

Zod schemas are common at boundaries:

- tRPC `.input(...)`
- booking body schemas
- feature form schemas
- DTO-like validation for shared types

Examples:

- `packages/features/bookings/lib/bookingCreateBodySchema.ts`
- `packages/trpc/server/routers/viewer/eventTypes/heavy/create.schema.ts`
- `packages/features/eventtypes/lib/schemas.ts`

## Testing conventions

### Unit and integration tests

Use Vitest for most packages.

Root config:

- `vitest.workspace.ts`
- `vitest.config.mts`

Find tests:

```powershell
rg --files | rg "\.(test|spec)\.(ts|tsx)$"
```

### E2E tests

Use Playwright.

Root config:

- `playwright.config.ts`

Run through scripts such as:

- `yarn test-e2e`

### API v2 tests

API v2 uses Jest through Nest tooling.

Look under:

- `apps/api/v2`

## Non-obvious conventions and gotchas

### Hybrid routing

`apps/web` uses both App Router and Pages Router. Always check both `apps/web/app` and `apps/web/pages` before assuming a route does not exist.

### Booking creation is special

Booking submit goes through `/api/book/event`, not tRPC.

Important path:

- `apps/web/pages/api/book/event.ts`

### Slot availability is computed

The slot endpoint combines many sources of truth. Do not look for a single final slots table.

Important path:

- `packages/trpc/server/routers/viewer/slots/util.ts`

### Some permission code may be intentionally stubbed in Cal.diy

In `packages/trpc/server/routers/viewer/eventTypes/heavy/create.handler.ts`, `PermissionCheckService` behavior is simplified/stubbed. Treat this as project-specific, and be careful when porting assumptions from upstream Cal.com behavior.

### Generated files exist

Generated Prisma/Zod/Kysely/OpenAPI/app-store outputs may be present. Do not hand-edit generated files.

### App-store code is broad

Integration behavior may be in a provider-specific folder under `packages/app-store`, while the domain flow starts in `packages/features`.

## What a good PR looks like

A PR that fits this repo usually:

- Changes the smallest domain area that owns the behavior.
- Uses existing package boundaries and import style.
- Adds/updates Zod schemas at API boundaries.
- Keeps route handlers thin.
- Puts multi-step business behavior in feature services.
- Uses repositories or existing data-access patterns.
- Updates tests close to the changed behavior.
- Documents new env vars in `.env.example`.
- Avoids unrelated refactors and generated churn.

Before opening a PR, ask:

- Did I update the right app/package?
- Did I validate input at the boundary?
- Did I preserve auth/public procedure behavior?
- Did I account for teams/orgs if the model supports them?
- Did I consider timezone behavior?
- Did I update the schema/migration if data shape changed?
