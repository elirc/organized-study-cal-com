# Glossary

These terms appear in file names, model names, variables, and product logic.

## Cal.diy

This project: a self-hostable scheduling application derived from the Cal.com codebase. It lets people create booking pages, define availability, connect calendars, and accept meetings.

## Booker

The public booking UI where a guest chooses date/time and submits their details.

Important file:

- `apps/web/modules/bookings/components/Booker.tsx`

## Organizer

The user who owns or hosts the event type being booked.

Related model:

- `User`

## Attendee

The guest who books or attends a meeting.

Related model:

- `Attendee`

## Event type

A bookable meeting template, such as "30 minute intro call".

Related model:

- `EventType`

Important code:

- `packages/features/eventtypes`
- `apps/web/modules/event-types`

## Booking

An actual scheduled meeting/reservation created from an event type.

Related model:

- `Booking`

Important code:

- `packages/features/bookings/lib/service/RegularBookingService.ts`
- `packages/features/bookings/lib/handleNewBooking/createBooking.ts`

## Schedule

A named set of availability rules, usually tied to a user and timezone.

Related model:

- `Schedule`

## Availability

Blocks of time when a user or event type can be booked.

Related model:

- `Availability`

## Slot

A computed available time that can be selected in the booker.

Important code:

- `packages/trpc/server/routers/viewer/slots/_router.tsx`
- `packages/trpc/server/routers/viewer/slots/util.ts`

Key idea:

- Slots are computed from schedules, availability, connected calendars, existing bookings, out-of-office entries, buffers, limits, and event type rules.

## Selected calendar

An external calendar that Cal.diy checks for conflicts.

Related model:

- `SelectedCalendar`

Example:

- "Check my Google Work calendar before showing slots."

## Destination calendar

The calendar where Cal.diy writes new booking events.

Related model:

- `DestinationCalendar`

Example:

- "Create new bookings on my Google Calendar."

## Credential

Encrypted credentials for an integration, such as calendar or conferencing access.

Related model:

- `Credential`

## App store

Integration system for calendars, video providers, CRMs, payments, messaging, and other external services.

Important directory:

- `packages/app-store`

## Booking fields

Custom form fields attached to an event type.

Related model field:

- `EventType.bookingFields`

## Responses

The answers an attendee submitted on the booking form.

Related model field:

- `Booking.responses`

## Custom inputs

Older/custom booking form input data.

Related model field:

- `Booking.customInputs`

## Recurring event

An event type or booking flow that creates a series of related bookings.

Related fields:

- `EventType.recurringEvent`
- `Booking.recurringEventId`

## Reschedule

Changing an existing booking's time. Reschedule flows carry references to the old booking and may cancel/replace old records transactionally.

Related fields:

- `Booking.rescheduled`
- `Booking.fromReschedule`
- `Booking.rescheduledBy`

Important code:

- `RegularBookingService.rescheduleBooking`

## Seated event

An event where multiple attendees can book seats in the same time slot.

Related models/fields:

- `BookingSeat`
- `EventType.seatsPerTimeSlot`

## Booking reference

External provider data connected to a booking, such as calendar event ID or conferencing URL.

Related model:

- `BookingReference`

## Round robin

A team scheduling mode where bookings are assigned across available hosts.

Related concepts:

- `Host`
- `schedulingType`
- host priority/weight behavior

## Collective event

A team event where multiple hosts may attend the same booking.

Related concepts:

- `Host`
- team `EventType`

## Managed event

An event type managed at team/organization level rather than purely personal ownership.

Related fields:

- `EventType.teamId`
- `EventType.parentId`
- managed event flags/settings

## Host

A user assigned to host a team event type.

Related model:

- `Host`

## Fixed host

A host that should be included or selected according to fixed-host rules rather than ordinary assignment.

Related concept:

- team event host assignment

## Host group

A grouping of hosts for advanced routing or assignment.

Related model:

- `HostGroup`

## Dynamic event type

A booking link that may involve a dynamic set of users/hosts rather than one static event owner.

Important loader:

- `apps/web/server/lib/[user]/[type]/getServerSideProps.ts`

## Hashed link

A private or obfuscated booking link for an event type.

Related model:

- `HashedLink`

## Profile

An org-aware public identity for a user.

Related model:

- `Profile`

## `upId`

An org/profile-aware identifier used in session/profile authorization logic.

Important file:

- `packages/features/auth/lib/userFromSessionUtils.ts`

## Team

A group of users that can have shared event types, members, roles, and branding.

Related model:

- `Team`

## Organization

A team-like container with organization behavior. In the schema, organizations are represented through `Team` with organization-specific fields/settings.

Related model:

- `Team`

## Membership

The relation between a user and a team, including role and accepted state.

Related model:

- `Membership`

## Webhook

An outbound HTTP callback fired when scheduling events happen.

Related model:

- `Webhook`

## Workflow

Automation around bookings, such as reminders or follow-up actions.

Important code areas:

- `packages/emails/workflow-email-service.ts`
- workflow-related Prisma migrations under `packages/prisma/migrations`

## Dry run booking

A booking attempt used to validate or price/check behavior without creating the final booking.

Related client handling:

- `apps/web/modules/bookings/hooks/useBookings.ts`

## Decoy booking

A bot/security-related response path where the client treats a booking-like response specially.

Related client handling:

- `apps/web/modules/bookings/hooks/useBookings.ts`

## Cal Video

The built-in video meeting option.

Related areas:

- event type Cal Video settings
- booking references
- app-store/provider code

## API v2

The public/platform API service under `apps/api/v2`, built with NestJS.

Use it for:

- external API contracts
- Swagger/OpenAPI
- platform-facing behavior

## tRPC viewer

The main internal RPC router for the web app.

Important file:

- `packages/trpc/server/routers/viewer/_router.tsx`

Example:

- `trpc.viewer.eventTypesHeavy.create.useMutation()`

## `publicProcedure`

tRPC wrapper for procedures that do not require login.

Important file:

- `packages/trpc/server/procedures/publicProcedure.ts`

## `authedProcedure`

tRPC wrapper for procedures that require login.

Important file:

- `packages/trpc/server/procedures/authedProcedure.ts`

## Idempotency key

A key used to avoid duplicate booking creation when requests retry.

Related field:

- `Booking.idempotencyKey`

## Booking audit

Audit trail for booking lifecycle changes.

Related model:

- `BookingAudit`

## Watchlist

Admin/security-oriented concept used in admin routes.

Search starting point:

```powershell
rg "watchlist"
```

## OOO

Out of office. Used to block availability and sometimes influence reassignment.

Related model:

- `OutOfOfficeEntry`

## Contact owner / routing

Routing-related booking metadata that can identify the owning contact or route a booking to a host.

Important mapper:

- `packages/features/bookings/lib/client/booking-event-form/booking-to-mutation-input-mapper.tsx`

## Creation source

Where a booking came from, such as web app or API.

Important usage:

- `apps/web/pages/api/book/event.ts` sets `creationSource: CreationSource.WEBAPP`.
