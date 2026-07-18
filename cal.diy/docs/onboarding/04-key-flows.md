# Key Flows

This file teaches you how to trace features end-to-end. The three most important flows are:

1. Booking a meeting
2. Creating an event type
3. Signing in and using authenticated tRPC

Each flow lists the important files in request order.

## Flow 1: Booking a meeting

This is the central user-facing flow. A guest opens a public booking link, chooses a date/time, fills the form, and submits a booking.

### Phase A: Render the public booking page

#### 1. `apps/web/app/(booking-page-wrapper)/[user]/[type]/page.tsx`

The App Router entry for a route like `/alice/intro-call`.

What it does:

- Receives route params for `[user]` and `[type]`.
- Calls the server-side loader through the app-dir compatibility wrapper.
- Renders the public event view.

Why it matters:

- This is the first file to open when debugging "the booking page does not load".

#### 2. `apps/web/server/lib/[user]/[type]/getServerSideProps.ts`

Server loader for the public event page.

What it does:

- Parses username and event type slug.
- Handles dynamic/group usernames.
- Checks the current session, if present.
- Handles organization routing and redirects.
- Calls `EventRepository.getPublicEvent`.
- Handles reschedule and seated-event context through helpers such as `getBookingForReschedule` and `getBookingForSeatedEvent`.
- Calculates branding visibility through `shouldHideBrandingForUserEvent`.

Magic to notice:

- Even though the route is in `app/`, this file follows a legacy `getServerSideProps` mental model.
- Public booking pages are not just "load event type by slug"; they also understand orgs, groups, reschedules, seats, and branding.

#### 3. `apps/web/modules/users/views/users-type-public-view.tsx`

The public user/event page view.

What it does:

- Receives server-loaded props.
- Renders the booker wrapper.

Why it matters:

- This is the bridge from route data into the booking UI module.

#### 4. `apps/web/modules/bookings/components/Booker.tsx`

The main booking UI.

What it does:

- Coordinates the booking page state machine.
- Shows date picker, time slots, booking form, verification dialog, and loading states.
- Moves through states like `loading`, `selecting_date`, `selecting_time`, and `booking`.
- Calls `handleBookEvent` when the final form is submitted.

Magic to notice:

- This component is a coordinator. Most business rules are not here; they are delegated to hooks, stores, tRPC, and feature packages.
- The Booker is shared conceptually with platform/embed flows, so some hooks live in `packages/platform/atoms`.

### Phase B: Find available slots

#### 5. `packages/trpc/server/routers/viewer/slots/_router.tsx`

tRPC router for slot-related procedures.

What it does:

- Registers procedures such as `getSchedule`, `reserveSlot`, `isAvailable`, and `removeSelectedSlotMark`.
- Uses `publicProcedure` for public slot reads.
- Validates input with schemas such as `ZGetScheduleInputSchema`.

#### 6. `packages/trpc/server/routers/viewer/slots/getSchedule.handler.ts`

Handler for slot schedule queries.

What it does:

- Receives validated tRPC input.
- Delegates to availability/slot calculation utilities.

#### 7. `packages/trpc/server/routers/viewer/slots/util.ts`

The heavy slot calculation utility.

Important class/function:

- `AvailableSlotsService`

What it does:

- Loads event type data.
- Resolves static and dynamic event types.
- Reads host/user schedules.
- Accounts for selected calendars and busy times.
- Accounts for existing bookings, reserved slots, out-of-office entries, booking limits, buffers, and restriction schedules.
- Produces available slots for the booker.

Magic to notice:

- Slots are computed at request time. There is no simple `slots` table that stores final availability.
- Short-lived slot caching exists through `withSlotsCache`; `SLOTS_CACHE_TTL` controls the TTL.
- "No slots" behavior can trigger notification logic in feature code.

### Phase C: Submit the booking form

#### 8. `apps/web/modules/bookings/components/BookEventForm/BookEventForm.tsx`

The final booking form.

What it does:

- Renders attendee fields through booking field components.
- Uses React Hook Form for form state.
- Submits guest responses, metadata, and booking fields.

Related file:

- `apps/web/modules/bookings/components/BookEventForm/BookingFields.tsx`

#### 9. `packages/platform/atoms/hooks/bookings/useHandleBookEvent.ts`

Submit orchestration hook.

Important function:

- `handleBookEvent`

What it does:

- Reads form values, selected time, timezone, duration, reschedule fields, recurring context, and metadata.
- Decides whether the submission is regular or recurring.
- Calls input mappers before invoking booking mutations.

Why it lives in `packages/platform`:

- Booking submission behavior is shared by web/platform/embed surfaces.

#### 10. `packages/features/bookings/lib/client/booking-event-form/booking-to-mutation-input-mapper.tsx`

Maps UI form state into the API request body.

Important functions:

- `mapBookingToMutationInput`
- `mapRecurringBookingToMutationInput`

What it does:

- Builds fields such as `user`, `start`, `end`, `eventTypeId`, `eventTypeSlug`, `timeZone`, `language`, `rescheduleUid`, `rescheduledBy`, `metadata`, `bookingUid`, `seatReferenceUid`, routing/contact owner fields, `_isDryRun`, `dub_id`, and `verificationCode`.

Magic to notice:

- If a field appears in the API body but not obviously in the form, check this mapper.

#### 11. `apps/web/modules/bookings/hooks/useBookings.ts`

Client mutation hook.

What it does:

- Creates TanStack Query mutations for regular and recurring booking creation.
- Calls `createBooking` or `createRecurringBooking`.
- Handles success states:
  - dry run
  - decoy booking
  - payment redirect
  - success redirect
  - embed SDK events
  - confirmation state

Magic to notice:

- The post-submit UX is mostly here, not in the API route.

#### 12. `packages/features/bookings/lib/create-booking.ts`

Client helper for the booking API call.

What it does:

- Posts the mapped body to `/api/book/event`.

Why it matters:

- Booking creation is not a tRPC mutation from the browser. It uses a normal Next API route.

### Phase D: API validation and service orchestration

#### 13. `apps/web/pages/api/book/event.ts`

Next API route for public booking creation.

What it does:

- Receives `POST /api/book/event`.
- Checks Turnstile if configured.
- Runs bot detection and IP-based rate limiting.
- Reads the optional NextAuth session.
- Sets `creationSource: CreationSource.WEBAPP`.
- Calls `getRegularBookingService().createBooking(...)`.
- Wraps the handler with `defaultResponder(handler, "/api/book/event")`.

Magic to notice:

- Public booking submissions can be unauthenticated, but session data is still included when available.
- Security checks happen before the booking service.

#### 14. `packages/features/bookings/di/RegularBookingService.container.ts`

Dependency injection container for booking service construction.

What it does:

- Builds the `RegularBookingService` with its dependencies.

Magic to notice:

- Booking code uses DI patterns from `@evyweb/ioctopus`. When you wonder "where did this service come from?", check the `di` folder.

#### 15. `packages/features/bookings/lib/service/RegularBookingService.ts`

The core booking orchestration service.

Important methods:

- `createBooking`
- `rescheduleBooking`
- `createBookingForApiV1`

What it does:

- Validates event type and reschedule restrictions.
- Builds booking data.
- Loads and validates users/hosts.
- Ensures users are available.
- Handles dynamic, team, recurring, seated, and payment-aware bookings.
- Builds calendar event data.
- Persists bookings through helper functions.
- Calls calendar/conferencing integrations.
- Fires emails, SMS, webhooks, workflow triggers, no-show triggers, analytics, and audit/reporting behavior.

Magic to notice:

- This file is intentionally orchestration-heavy. Do not expect a booking to be "just one insert".
- When debugging booking behavior, identify which phase you are in: validation, availability, persistence, external calendar creation, notification, or response handling.

#### 16. `packages/features/bookings/lib/bookingCreateBodySchema.ts`

Booking request schema.

Important exports:

- `bookingCreateBodySchema`
- `extendedBookingCreateBody`

What it does:

- Defines and validates the shape of the booking create body.

Why it matters:

- If a client field is rejected or missing server-side, check this schema and the mapper together.

### Phase E: Persistence and side effects

#### 17. `packages/features/bookings/lib/handleNewBooking/createBooking.ts`

Persistence helper for new bookings.

Important functions:

- `_createBooking`
- `createBooking`
- `buildNewBookingData`
- `saveBooking`

What it does:

- Builds Prisma data for the booking.
- Creates `Booking`, `Attendee`, tracking, seats, references, and related records.
- Handles cancellation of an old booking in reschedule transactions.
- Wraps creation with reporting through `withReporting`.

#### 18. `packages/prisma/schema.prisma`

Database model definitions.

Important models for this flow:

- `Booking`
- `Attendee`
- `BookingReference`
- `BookingSeat`
- `Payment`
- `EventType`
- `User`
- `Schedule`
- `Availability`
- `SelectedCalendar`
- `DestinationCalendar`
- `Credential`

#### 19. `packages/app-store/**`

Integration implementations.

What it does:

- Calendar integrations create/update external calendar events.
- Conferencing integrations create meeting links.
- Payment integrations handle paid bookings.

## Flow 2: Creating an event type

This is the main logged-in configuration flow. A user creates a bookable event such as "30 minute meeting".

### 1. `apps/web/modules/event-types/components/CreateEventTypeDialog.tsx`

Dialog component for creating an event type.

What it does:

- Opens the creation UI.
- Renders `CreateEventTypeForm`.
- Calls `useCreateEventType`.
- On success, routes to `/event-types/${eventType.id}`.
- Shows toast feedback.

#### 2. `packages/features/eventtypes/components/CreateEventTypeForm.tsx`

Shared form component.

What it does:

- Renders fields like title, slug, description, and length.
- Uses rich text conversion through `Editor`, `md`, and `turndown`.
- Slugifies the title until the slug has been manually touched.
- Uses constants such as `MIN_EVENT_DURATION_MINUTES` and `MAX_EVENT_DURATION_MINUTES`.

Magic to notice:

- UI components for event types are split between `apps/web/modules/event-types` and `packages/features/eventtypes`. Shared/domain-heavy components live in the feature package.

#### 3. `apps/web/modules/event-types/hooks/useCreateEventType.ts`

Mutation hook.

What it does:

- Uses form behavior from `@calcom/atoms/hooks/event-types/private/useCreateEventTypeForm`.
- Calls `trpc.viewer.eventTypesHeavy.create.useMutation`.
- On success, invalidates/fetches event type lists such as `viewer.eventTypes.getEventTypesFromGroup`.
- Handles errors:
  - `BAD_REQUEST` for duplicate event type slug
  - `UNAUTHORIZED` for permission failures

#### 4. `packages/trpc/server/routers/viewer/eventTypes/heavy/_router.ts`

tRPC router for heavier event type mutations.

What it does:

- Registers `create` as an authenticated mutation.
- Uses `authedProcedure`.
- Validates input through `ZCreateInputSchema`.
- Delegates to `create.handler`.

#### 5. `packages/trpc/server/routers/viewer/eventTypes/heavy/create.schema.ts`

Input schema for the tRPC mutation.

What it does:

- Exports `ZCreateInputSchema`.
- Reuses `createEventTypeInput` from feature schemas.

#### 6. `packages/features/eventtypes/lib/schemas.ts`

Feature-level event type schemas.

Important export:

- `createEventTypeInput`

What it does:

- Defines the domain-level shape of event type creation input.

#### 7. `packages/trpc/server/routers/viewer/eventTypes/heavy/create.handler.ts`

Server handler for event type creation.

Important function:

- `createHandler`

What it does:

- Reads `ctx.user.id`.
- Extracts fields such as `schedulingType`, `teamId`, `metadata`, `locations`, `scheduleId`, and `calVideoSettings`.
- Checks permissions for team event creation.
- Applies default locations through `getDefaultLocations(ctx.user)`.
- Builds Prisma-compatible `EventTypeCreateInput`.
- Handles org profile behavior.
- Calls `new EventTypeRepository(ctx.prisma).create(...)`.
- Converts unique slug conflicts into `BAD_REQUEST`.

Magic to notice:

- In this Cal.diy codebase, `PermissionCheckService` is stubbed to allow true in this handler. That is non-standard and worth calling out in reviews.
- Team and organization behavior make this more complex than a simple personal event type insert.

#### 8. `packages/features/eventtypes/repositories/eventTypeRepository.ts`

Repository for event type persistence.

Important class:

- `EventTypeRepository`

Important methods:

- `create`
- `generateCreateEventTypeData`

What it does:

- Converts service/handler data into Prisma create data.
- Calls `prismaClient.eventType.create`.
- Includes `calVideoSettings` in returned data.

#### 9. `packages/prisma/schema.prisma`

Important model:

- `EventType`

Important relations:

- Optional owner through `userId`.
- Optional team through `teamId`.
- Optional org-aware profile through `profileId`.
- Related bookings, availability, selected calendars, webhooks, destination calendar, hosts, hashed links, and Cal Video settings.

## Flow 3: Signing in and using authenticated tRPC

This flow explains how a user logs in and how later tRPC requests know who they are.

### Phase A: Login page and credential submit

#### 1. `apps/web/app/(use-page-wrapper)/auth/login/page.tsx`

App Router entry for login.

What it does:

- Loads server-side props for the login page.
- Renders the login view.

#### 2. `apps/web/modules/auth/login-view.tsx`

Login UI.

What it does:

- Renders email/password login.
- Calls `signIn("credentials", { ...values, callbackUrl, redirect: false })`.
- Supports Google and Azure AD sign-in through `signIn("google")` and `signIn("azure-ad")` when enabled.
- Handles two-factor auth and backup code paths.
- Uses `getSafeRedirectUrl` for redirect safety.

Magic to notice:

- The UI does not validate the password itself. It delegates to NextAuth's credentials provider.

### Phase B: NextAuth route and auth options

#### 3. `apps/web/pages/api/auth/[...nextauth].ts`

NextAuth catch-all API route.

What it does:

- Calls `NextAuth(req, res, getOptions(...))`.
- Injects helpers for tracking/Dub data.

#### 4. `packages/features/auth/lib/next-auth-options.ts`

Core NextAuth configuration.

Important exports/functions:

- `getOptions`
- `authorizeCredentials`
- `CalComCredentialsProvider`

What it does:

- Configures providers: credentials, email, Google, Azure AD depending on env.
- Uses JWT sessions.
- Defines callbacks for `jwt`, `session`, `signIn`, and `redirect`.
- Checks user password, locked state, rate limits, two-factor auth, backup codes, and inactive admin behavior.
- Reads users through auth repositories.

Magic to notice:

- Credentials auth is project-specific and much richer than a basic email/password check.
- Provider availability depends on environment variables.

### Phase C: Session lookup for server code

#### 5. `packages/features/auth/lib/userFromSessionUtils.ts`

Session-to-user helper.

Important function:

- `getUserSession`

What it does:

- Calls NextAuth session helpers.
- Reloads the user from the database.
- Enriches session data with profile information.
- Validates profile authorization.
- Returns the user/session shape used by tRPC and server code.

### Phase D: tRPC context and auth middleware

#### 6. `packages/trpc/server/createContext.ts`

tRPC request context creation.

What it does:

- Adds `prisma` and `readonlyPrisma`.
- Adds session/user info when available.
- Adds request metadata such as `sourceIp`, `locale`, and trace context.

Why it matters:

- Every tRPC handler receives `ctx`. If you need user, database, locale, or request metadata, this is where it enters.

#### 7. `packages/trpc/server/middlewares/sessionMiddleware.ts`

Auth middleware for protected procedures.

Important middleware:

- `isAuthed`

What it does:

- Calls `getUserSession`.
- Throws `TRPCError` with `UNAUTHORIZED` if there is no valid user.
- Sets Sentry user context.
- Passes an authenticated user into downstream handlers.

#### 8. `packages/trpc/server/procedures/authedProcedure.ts`

Protected tRPC procedure wrapper.

What it does:

- Applies performance middleware.
- Applies error conversion middleware.
- Applies `isAuthed`.

Use it when:

- A tRPC endpoint requires a logged-in user.

#### 9. `packages/trpc/server/procedures/publicProcedure.ts`

Public tRPC procedure wrapper.

What it does:

- Applies common performance and error middleware.
- Does not require auth.

Use it when:

- A tRPC endpoint can be called without login, such as public slot reads.

#### 10. `packages/trpc/server/middlewares/errorConversionMiddleware.ts`

Error conversion middleware.

What it does:

- Converts project-specific errors into tRPC-friendly errors.

Magic to notice:

- Service code may throw project errors such as `ErrorWithCode`; tRPC middleware is responsible for turning those into API responses.

### Example authenticated tRPC request

Creating an event type after login:

```text
apps/web/modules/event-types/hooks/useCreateEventType.ts
  -> trpc.viewer.eventTypesHeavy.create.useMutation()
packages/trpc/server/routers/viewer/eventTypes/heavy/_router.ts
  -> authedProcedure.input(ZCreateInputSchema).mutation(...)
packages/trpc/server/procedures/authedProcedure.ts
  -> isAuthed middleware
packages/features/auth/lib/userFromSessionUtils.ts
  -> getUserSession()
packages/trpc/server/routers/viewer/eventTypes/heavy/create.handler.ts
  -> createHandler({ ctx, input })
packages/features/eventtypes/repositories/eventTypeRepository.ts
  -> EventTypeRepository.create(...)
packages/prisma/schema.prisma
  -> EventType row
```

## How to trace a new flow

Use this checklist:

1. Find the route or component with `rg`.
2. Identify whether the submit/read path is tRPC, Next API, or API v2.
3. Follow schemas before handlers.
4. Follow handlers into feature services.
5. Follow services into repositories or Prisma calls.
6. Check `packages/prisma/schema.prisma` for model relationships.
7. Check side effects: emails, webhooks, app-store integrations, analytics, queues.

Useful commands:

```powershell
rg "viewer.eventTypesHeavy.create"
rg "createBooking"
rg "model Booking" packages/prisma/schema.prisma
rg "authedProcedure" packages/trpc/server/routers/viewer
```
