# Onboarding Journal

This is the narrative version of the onboarding process. The point is not only to say what the repo contains, but to model how a senior engineer would build a map of it.

## Day 1 instinct: find the control room

When I open a repo like this, I do not start with random source files. I start with the files that tell me how the repo thinks about itself:

```powershell
Get-ChildItem
Get-Content package.json
Get-Content turbo.json
Get-Content .env.example
rg --files -g package.json
```

For Cal.diy, the first big signal is the root `package.json`. It says this is a Yarn 4 monorepo with workspaces under `apps/*`, `packages/*`, `packages/features/*`, `packages/app-store/*`, `packages/platform/*`, and `example-apps/*`.

That immediately gives me a first mental model:

```text
apps = things you can run
packages = things apps depend on
packages/features = product/domain behavior
packages/prisma = database truth
packages/trpc = internal web API
```

Then `turbo.json` tells me how the repo is operated. If a task is hard to understand, I check whether Turborepo is composing multiple package scripts behind one root command.

## Day 1 instinct: find the product's main verb

For a scheduling app, the main verb is "book".

So I search for booking entry points:

```powershell
rg "createBooking"
rg "Booker"
rg "/api/book/event"
```

That leads to a good vertical slice:

- `apps/web/app/(booking-page-wrapper)/[user]/[type]/page.tsx`
- `apps/web/server/lib/[user]/[type]/getServerSideProps.ts`
- `apps/web/modules/bookings/components/Booker.tsx`
- `apps/web/modules/bookings/hooks/useBookings.ts`
- `packages/features/bookings/lib/create-booking.ts`
- `apps/web/pages/api/book/event.ts`
- `packages/features/bookings/lib/service/RegularBookingService.ts`
- `packages/features/bookings/lib/handleNewBooking/createBooking.ts`
- `packages/prisma/schema.prisma`

This is the thread I would give a junior developer first. It crosses route, server load, UI state, form submit, API transport, service orchestration, persistence, and database models.

The important discovery is that booking creation is not a tRPC mutation. It posts to a Next API route:

```text
/api/book/event
```

That is the kind of fact that prevents hours of searching under `packages/trpc` for the wrong endpoint.

## Day 1 instinct: separate transport from domain

Once I find `apps/web/pages/api/book/event.ts`, I ask: "Is this where the business logic lives?"

The answer is mostly no. It handles request-level concerns:

- bot checks
- rate limits
- session lookup
- `CreationSource.WEBAPP`
- calling `getRegularBookingService().createBooking(...)`

The business logic is in:

- `packages/features/bookings/lib/service/RegularBookingService.ts`

That file is large because the booking domain is large. It is responsible for validating event types, users, availability, payment, reschedules, recurring events, calendar event construction, persistence, external integrations, notifications, webhooks, and analytics.

The lesson: do not judge this flow by the API route alone. The route is only the door.

## Day 2 instinct: find the API style

Next I want to know how the app usually talks to the backend.

I search:

```powershell
rg "createTRPCReact"
rg "viewerRouter"
rg "authedProcedure"
```

That leads to:

- `apps/web/app/_trpc/trpc.ts`
- `apps/web/app/_trpc/trpc-client.ts`
- `apps/web/app/_trpc/trpc-provider.tsx`
- `packages/trpc/server/routers/_app.ts`
- `packages/trpc/server/routers/viewer/_router.tsx`
- `packages/trpc/server/procedures/authedProcedure.ts`
- `packages/trpc/server/procedures/publicProcedure.ts`

The tRPC mental model is:

```text
React component/hook
  -> trpc.viewer.area.procedure.useQuery/useMutation
  -> viewer router
  -> schema
  -> handler
  -> feature service/repository
  -> Prisma
```

I verify that model by tracing event type creation:

- UI opens in `apps/web/modules/event-types/components/CreateEventTypeDialog.tsx`.
- Hook calls `trpc.viewer.eventTypesHeavy.create.useMutation()` in `apps/web/modules/event-types/hooks/useCreateEventType.ts`.
- Router registers it in `packages/trpc/server/routers/viewer/eventTypes/heavy/_router.ts`.
- Schema comes from `packages/trpc/server/routers/viewer/eventTypes/heavy/create.schema.ts`.
- Handler is `packages/trpc/server/routers/viewer/eventTypes/heavy/create.handler.ts`.
- Persistence goes through `packages/features/eventtypes/repositories/eventTypeRepository.ts`.
- Model is `EventType` in `packages/prisma/schema.prisma`.

This confirms the normal tRPC pattern and gives me a comparison point for booking creation, which is special.

## Day 2 instinct: understand auth before changing protected flows

When I see `authedProcedure`, I follow the user/session path:

```powershell
rg "isAuthed"
rg "getUserSession"
rg "NextAuth"
```

That leads to:

- `apps/web/pages/api/auth/[...nextauth].ts`
- `packages/features/auth/lib/next-auth-options.ts`
- `packages/features/auth/lib/userFromSessionUtils.ts`
- `packages/trpc/server/createContext.ts`
- `packages/trpc/server/middlewares/sessionMiddleware.ts`
- `packages/trpc/server/procedures/authedProcedure.ts`

The key point is that authenticated tRPC is not just "trust the JWT". The app reloads/enriches session user data and applies profile-aware authorization.

When a protected mutation has surprising `UNAUTHORIZED` behavior, I would inspect:

1. The frontend call.
2. Whether the server uses `authedProcedure`.
3. `isAuthed` in `sessionMiddleware.ts`.
4. `getUserSession` in `userFromSessionUtils.ts`.
5. Any feature-specific permission checks in the handler.

## Day 3 instinct: read the database as a product document

`packages/prisma/schema.prisma` is long, but it tells the product story.

I do not try to memorize every model. I group them:

```text
Identity:
  User, Profile, Team, Membership

Scheduling setup:
  EventType, Host, Schedule, Availability,
  SelectedCalendar, DestinationCalendar

Booking result:
  Booking, Attendee, BookingReference,
  BookingSeat, Payment, BookingAudit

Integrations/platform:
  Credential, App, Webhook, ApiKey,
  OAuthClient, AccessToken, RefreshToken
```

Then I connect the models to flows.

For "why is a slot available?", I inspect:

- `EventType`
- `Schedule`
- `Availability`
- `SelectedCalendar`
- `Booking`
- `OutOfOfficeEntry`

For "why did booking creation fail?", I inspect:

- `EventType`
- `User`
- `Booking`
- `Attendee`
- `Credential`
- `DestinationCalendar`
- `BookingReference`

The schema is not just storage. It is the product map.

## Day 3 instinct: look for team rules

Before adding code, I read repo rules:

- `agents/rules`
- `CONTRIBUTING.md`

The patterns that matter most:

- keep PRs small and focused
- prefer feature boundaries
- prefer repositories for database access
- prefer Prisma `select` over broad `include`
- use `ErrorWithCode` in services/repositories and `TRPCError` in tRPC surfaces
- keep route handlers thin
- name services and repositories consistently

This tells me what maintainers will expect in review.

## Day 4 instinct: find one weird thing and write it down

Every mature codebase has local surprises. In Cal.diy, a few stand out:

- `apps/web` is hybrid App Router and Pages Router.
- Booking creation uses `/api/book/event`, not tRPC.
- Slots are computed from many inputs, not read from a final slots table.
- Some permission code in `packages/trpc/server/routers/viewer/eventTypes/heavy/create.handler.ts` is simplified/stubbed in this Cal.diy codebase.
- Some older code calls Prisma directly even though repository classes are the preferred direction.

These are the notes I would keep open while reviewing or mentoring. New developers usually get stuck on exceptions to the pattern, not the pattern itself.

## My personal workflow for a new feature

If I were asked to change a feature, I would work in this order:

1. Find the user-facing route/component.
2. Identify the transport: tRPC, Next API, or API v2.
3. Read the input schema.
4. Read the handler.
5. Read the feature service/repository.
6. Read the relevant Prisma models.
7. Search for tests near the changed code.
8. Make the smallest change that fits the existing local style.
9. Run the narrowest useful tests first, then broader checks if the change crosses boundaries.

Example for event type creation:

```powershell
rg "eventTypesHeavy.create"
rg "createEventTypeInput"
rg "EventTypeRepository"
rg "model EventType" packages/prisma/schema.prisma
```

Example for booking:

```powershell
rg "/api/book/event"
rg "RegularBookingService"
rg "bookingCreateBodySchema"
rg "model Booking" packages/prisma/schema.prisma
```

## What I would ask a junior developer to do first

I would ask them to trace one successful booking from the browser to the database.

Deliverable:

- Write down the request body created by `mapBookingToMutationInput`.
- Find where `/api/book/event` receives it.
- Find where `RegularBookingService.createBooking` starts.
- Find where `Booking` and `Attendee` are written.
- Open Prisma Studio and identify the created rows.

This task builds the right instincts:

- route tracing
- API boundary tracing
- schema awareness
- service orchestration awareness
- database relationship awareness

After that, the repo starts feeling much smaller.
