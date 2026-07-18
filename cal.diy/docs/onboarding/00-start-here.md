# Start Here

Cal.diy is a scheduling application: users create event types, define availability, connect calendars, and share booking links. Guests use those links to pick a slot and create a booking, while the system checks calendars, writes records, creates external references, and sends notifications.

The repo is a Yarn/Turborepo monorepo. The main app is `apps/web`, domain behavior lives mostly in `packages/features`, internal web APIs live in `packages/trpc`, and the database source of truth is `packages/prisma/schema.prisma`.

## The five files/folders to understand first

### `apps/web/`

The main Next.js product app. It contains public booking pages, logged-in dashboards, auth pages, settings, app routes, legacy pages, and API routes.

Start here when you are working on user-visible web behavior.

### `packages/features/`

The domain layer. Booking logic, event type logic, auth helpers, schedules, availability, workflows, teams, and other product behavior live here.

Start here when you need to understand what the product means, not just how a route renders.

### `packages/trpc/server/routers/viewer/_router.tsx`

The main internal API map for the web app. Most `trpc.viewer.*` calls are registered under this router.

Start here when tracing logged-in app mutations and queries.

### `packages/features/bookings/lib/service/RegularBookingService.ts`

The central booking orchestration service. It validates booking requests, checks availability, handles reschedules/seats/payments, persists bookings, and triggers integrations/notifications.

Start here when debugging booking creation behavior.

### `packages/prisma/schema.prisma`

The database source of truth. It defines `User`, `EventType`, `Schedule`, `Availability`, `SelectedCalendar`, `Booking`, `Attendee`, `Credential`, `Team`, and the rest of the product data model.

Start here when a feature question turns into "what data exists and how is it related?"

## Recommended reading order

1. `docs/onboarding/01-repo-map.md`
2. `docs/onboarding/02-tech-stack.md`
3. `docs/onboarding/03-architecture.md`
4. `docs/onboarding/04-key-flows.md`
5. `docs/onboarding/05-data-models.md`
6. `docs/onboarding/06-patterns.md`
7. `docs/onboarding/07-local-dev.md`
8. `docs/onboarding/08-glossary.md`
9. `docs/onboarding/09-onboarding-journal.md`

## Suggested first task

Trace how a booking is created.

Start at:

- `apps/web/app/(booking-page-wrapper)/[user]/[type]/page.tsx`

Then follow:

- `apps/web/server/lib/[user]/[type]/getServerSideProps.ts`
- `apps/web/modules/bookings/components/Booker.tsx`
- `apps/web/modules/bookings/components/BookEventForm/BookEventForm.tsx`
- `packages/platform/atoms/hooks/bookings/useHandleBookEvent.ts`
- `packages/features/bookings/lib/client/booking-event-form/booking-to-mutation-input-mapper.tsx`
- `apps/web/modules/bookings/hooks/useBookings.ts`
- `packages/features/bookings/lib/create-booking.ts`
- `apps/web/pages/api/book/event.ts`
- `packages/features/bookings/lib/service/RegularBookingService.ts`
- `packages/features/bookings/lib/handleNewBooking/createBooking.ts`
- `packages/prisma/schema.prisma`

By the end of that trace, you will understand the repo's most important pattern: user action to UI state to API boundary to feature service to database to integrations and back to the browser.
