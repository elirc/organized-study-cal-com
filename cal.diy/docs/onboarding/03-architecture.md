# Architecture Overview

Cal.diy is a scheduling platform. Users create event types, connect calendars, define availability, and share booking links. Guests pick a slot and submit a booking. The system checks availability, writes booking records, creates calendar/conferencing references, and sends notifications.

## Is this a monorepo?

Yes. This is a Yarn/Turborepo monorepo.

High-level shape:

```text
cal.diy
|-- apps
|   |-- web        Main Next.js scheduling app
|   |-- api        Public API services, especially apps/api/v2
|   `-- docs       Documentation site
|-- packages
|   |-- features   Domain behavior: bookings, event types, auth, schedules, etc.
|   |-- trpc       Internal typed RPC API for the web app
|   |-- prisma     Database schema, Prisma client, migrations, seeds
|   |-- app-store  Third-party integrations
|   |-- platform   Platform hooks, atoms, SDK-style packages
|   |-- ui         Shared React UI
|   `-- lib        Shared utilities
`-- docs
    `-- onboarding Developer onboarding docs
```

## Main apps

### `apps/web`

The primary product. It handles:

- Public booking pages
- Logged-in dashboard
- Event type management
- Auth pages
- Settings
- Team/org flows
- Next API routes
- tRPC HTTP endpoints

It uses both Next.js App Router and Pages Router. A common pattern is:

1. App Router page receives the route.
2. A server loader under `apps/web/server/lib` fetches initial data.
3. A module under `apps/web/modules` renders the UI.
4. Mutations go through tRPC or a Next API route.

### `apps/api/v2`

The public API v2 service. It is a NestJS app with controllers, modules, services, guards, Swagger/OpenAPI, queues, Redis support, and API tests.

Use this when working on external/public API behavior rather than the internal web app RPC layer.

### `apps/docs`

The docs site. It is separate from the app and API runtime.

## Main packages

### `packages/features`

Domain logic. This is where "calendar scheduling behavior" lives.

Examples:

- `packages/features/bookings/lib/service/RegularBookingService.ts`
- `packages/features/bookings/lib/handleNewBooking/createBooking.ts`
- `packages/features/eventtypes/components/CreateEventTypeForm.tsx`
- `packages/features/eventtypes/repositories/eventTypeRepository.ts`
- `packages/features/auth/lib/next-auth-options.ts`

### `packages/trpc`

Internal typed API for the web app.

The root router is `packages/trpc/server/routers/_app.ts`, which exposes `viewer` from `packages/trpc/server/routers/viewer/_router.tsx`.

Typical flow:

```text
React hook -> trpc.viewer.someArea.someProcedure.useMutation()
           -> packages/trpc/server/routers/viewer/someArea/*
           -> feature service/repository
           -> Prisma/Postgres
```

### `packages/prisma`

Database source of truth.

The schema is `packages/prisma/schema.prisma`. It defines users, profiles, teams, memberships, event types, schedules, availability, bookings, attendees, credentials, selected calendars, destination calendars, webhooks, OAuth clients, API keys, platform tokens, and more.

`packages/prisma/index.ts` exports Prisma clients used by services and tRPC context.

### `packages/app-store`

Third-party app and integration implementations: calendars, video providers, CRMs, payments, analytics, messaging, and more.

Booking creation eventually talks to app-store integrations through calendar/event manager code to create external calendar events or meeting links.

## How frontend talks to backend

There are three important communication paths.

### 1. tRPC for internal web app APIs

Used by logged-in app features and some public data reads.

Important files:

- Client creation: `apps/web/app/_trpc/trpc-client.ts`
- Provider: `apps/web/app/_trpc/trpc-provider.tsx`
- Root router: `packages/trpc/server/routers/_app.ts`
- Viewer router: `packages/trpc/server/routers/viewer/_router.tsx`
- Request context: `packages/trpc/server/createContext.ts`
- Public procedure: `packages/trpc/server/procedures/publicProcedure.ts`
- Auth procedure: `packages/trpc/server/procedures/authedProcedure.ts`

Example:

```text
apps/web/modules/event-types/hooks/useCreateEventType.ts
  calls trpc.viewer.eventTypesHeavy.create.useMutation()
packages/trpc/server/routers/viewer/eventTypes/heavy/_router.ts
  registers create
packages/trpc/server/routers/viewer/eventTypes/heavy/create.handler.ts
  validates auth and builds create data
packages/features/eventtypes/repositories/eventTypeRepository.ts
  writes EventType through Prisma
packages/prisma/schema.prisma
  defines EventType
```

### 2. Next API routes for special web endpoints

Booking creation is the most important example.

Important files:

- `apps/web/pages/api/book/event.ts`
- `packages/features/bookings/lib/create-booking.ts`
- `packages/features/bookings/lib/service/RegularBookingService.ts`

The public booker posts to `/api/book/event`, then the API route calls the booking service.

### 3. NestJS API v2 for public/platform API

API v2 lives under `apps/api/v2`. This is a separate application from the Next.js web app.

## Architecture diagram

```text
                         +----------------------+
                         |      Browser UI      |
                         | apps/web modules     |
                         +----------+-----------+
                                    |
               +--------------------+--------------------+
               |                                         |
               v                                         v
    +----------------------+                +-------------------------+
    | tRPC client/provider |                | Next API route          |
    | apps/web/app/_trpc   |                | /api/book/event         |
    +----------+-----------+                +------------+------------+
               |                                         |
               v                                         v
    +----------------------+                +-------------------------+
    | tRPC routers         |                | RegularBookingService   |
    | packages/trpc/server |                | packages/features       |
    +----------+-----------+                +------------+------------+
               |                                         |
               v                                         v
    +----------------------+                +-------------------------+
    | Feature handlers     |<-------------->| Domain services/repos   |
    | packages/features    |                | packages/features       |
    +----------+-----------+                +------------+------------+
               |                                         |
               +--------------------+--------------------+
                                    |
                                    v
                         +----------------------+
                         | Prisma/Postgres      |
                         | packages/prisma      |
                         +----------+-----------+
                                    |
                                    v
                         +----------------------+
                         | App integrations     |
                         | packages/app-store   |
                         +----------------------+
```

## Full data path: booking a meeting

The booking flow is a good end-to-end example because it crosses frontend UI, API validation, service orchestration, database writes, external integrations, and notifications.

### 1. User opens a booking link

Route:

- `apps/web/app/(booking-page-wrapper)/[user]/[type]/page.tsx`

Server data:

- `apps/web/server/lib/[user]/[type]/getServerSideProps.ts`

The server loader parses the username and event type slug, handles dynamic/group bookings, checks session context, fetches the public event through `EventRepository.getPublicEvent`, and prepares props for the booking page.

### 2. UI renders the booker

View:

- `apps/web/modules/users/views/users-type-public-view.tsx`

Main UI:

- `apps/web/modules/bookings/components/Booker.tsx`

The booker tracks a state machine: loading, selecting date, selecting time, and filling the booking form.

### 3. User selects a slot

The UI queries slots through tRPC:

- `packages/trpc/server/routers/viewer/slots/_router.tsx`
- `packages/trpc/server/routers/viewer/slots/getSchedule.handler.ts`
- `packages/trpc/server/routers/viewer/slots/util.ts`

`AvailableSlotsService` combines event type rules, schedules, selected calendars, existing bookings, out-of-office entries, buffers, limits, and reserved slots to produce available times.

### 4. User submits booking form

Form:

- `apps/web/modules/bookings/components/BookEventForm/BookEventForm.tsx`

Submit handler:

- `packages/platform/atoms/hooks/bookings/useHandleBookEvent.ts`

Input mapping:

- `packages/features/bookings/lib/client/booking-event-form/booking-to-mutation-input-mapper.tsx`

Client mutation:

- `apps/web/modules/bookings/hooks/useBookings.ts`
- `packages/features/bookings/lib/create-booking.ts`

The form values are converted into the API body, including attendee responses, event type ID/slug, time zone, start/end time, metadata, reschedule fields, verification code, routing/contact-owner fields, and dry-run flags.

### 5. API route receives request

Endpoint:

- `apps/web/pages/api/book/event.ts`

This route checks bot protections/rate limits, reads the optional user session, sets `creationSource: CreationSource.WEBAPP`, and calls:

- `getRegularBookingService().createBooking(...)`

### 6. Booking service orchestrates domain behavior

Main service:

- `packages/features/bookings/lib/service/RegularBookingService.ts`

This is the core orchestration layer. It validates the event type, reschedule rules, host/user availability, payment, booking metadata, calendar event data, seats, recurring bookings, and integration behavior.

### 7. Booking is persisted

Persistence:

- `packages/features/bookings/lib/handleNewBooking/createBooking.ts`

This creates the `Booking`, `Attendee`, `BookingReference`, `Payment`, `BookingSeat`, tracking, and related records as needed. The model definitions live in:

- `packages/prisma/schema.prisma`

### 8. Side effects run

The booking service and event manager trigger side effects:

- Calendar event creation through app-store integrations
- Conferencing link creation
- Email/SMS notifications
- Webhooks
- Workflow triggers
- Analytics
- No-show and audit/reporting records

### 9. Response returns to browser

Client success handling lives in:

- `apps/web/modules/bookings/hooks/useBookings.ts`

It handles dry runs, decoy bookings, payment redirects, success redirects, embed SDK events, and the confirmation state.

## High-level database shape

The schema is large, but the scheduling core is understandable as a few clusters.

### Identity and organization

- `User`: a person with login, profile, timezone, event types, schedules, credentials, calendars, and bookings.
- `Profile`: org-aware public identity for a user.
- `Team`: team or organization container.
- `Membership`: connects users to teams with roles.

### Scheduling configuration

- `EventType`: bookable thing, such as "30 minute intro call".
- `Host`: connects users to team event types and controls host-specific behavior.
- `Schedule`: named availability schedule.
- `Availability`: individual availability blocks.
- `SelectedCalendar`: calendars checked for conflicts.
- `DestinationCalendar`: calendar where new bookings are written.

### Booking records

- `Booking`: the meeting reservation.
- `Attendee`: guest on a booking.
- `BookingReference`: external calendar/conferencing references.
- `Payment`: payment record for paid bookings.
- `BookingSeat`: seat record for seated events.
- `BookingAudit` and `BookingReport`: auditing/reporting.

### Integrations and platform

- `Credential`: encrypted integration credentials.
- `App`: app-store integration metadata.
- `Webhook`: outbound event hooks.
- `ApiKey`, `OAuthClient`, `AccessToken`, `RefreshToken`: API/platform access.

## The most important architectural surprises

### Booking creation is not tRPC

Many app operations use tRPC, but public booking creation posts to `apps/web/pages/api/book/event.ts`.

### The web app is hybrid Next.js

Expect both:

- App Router pages in `apps/web/app`
- Pages Router API routes and legacy pages in `apps/web/pages`

### Domain code is split from API transport

tRPC handlers and Next API routes should usually call feature services/repositories rather than carrying business logic directly.

### Availability is computed, not stored as final slots

Slots are calculated from schedules, event type rules, connected calendars, existing bookings, selected calendars, OOO entries, buffers, limits, and reserved slots.

### Prisma schema is the truth, but generated code is everywhere

Do not edit generated Prisma/Zod/Kysely output by hand. Change `packages/prisma/schema.prisma` and regenerate/migrate through the project scripts.
