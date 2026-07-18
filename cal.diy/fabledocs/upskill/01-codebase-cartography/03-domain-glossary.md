# Domain Glossary

Scheduling nouns and where they live in code. When an interviewer or a maintainer says one of these, you should be able to point at the model or file. Anchors as of 2026-07-11.

| Term | Means | Code location |
| --- | --- | --- |
| **Event Type** | A bookable meeting template (duration, availability, questions) | `EventType` model `schema.prisma:156`; router `viewer/eventTypes` |
| **Booking** | A confirmed/pending scheduled meeting instance | `Booking` model `schema.prisma:851`; `RegularBookingService.ts` |
| **Attendee** | A person on a booking (booker + guests) | `Attendee` model `schema.prisma:826` |
| **Availability** | A recurring weekly time rule (days + start/end) | `Availability` model `schema.prisma:960` |
| **Schedule** | A named set of availabilities in a timezone | `Schedule` model `schema.prisma:945` |
| **Slot** | A single bookable time; **computed, never stored** | derived in `slots/util.ts` from schedule − busy |
| **Busy time** | An interval a host is unavailable (external cal or existing booking) | `getBusyTimes`, `checkForConflicts.ts` |
| **Selected slot / Reservation** | A soft, expiring hold during checkout | `SelectedSlots` model `schema.prisma:1437`; `reserveSlot.handler.ts` |
| **Booking fields** | Per-event-type custom questions on the form | `EventType.bookingFields` (Json) `schema.prisma:193`; `form-builder/` |
| **Responses** | The booker's answers to booking fields | `Booking.responses` (Json) `schema.prisma:867` |
| **Reschedule** | Move an existing booking to a new time (cancels + recreates) | `RegularBookingService.ts:435-484`; `EventManager.reschedule` `:615` |
| **Seats / seated event** | Multiple attendees share one time slot | `EventType.seatsPerTimeSlot`; `BookingSeat` `schema.prisma:1330`; `handleSeats/` |
| **Round robin (RR)** | Team scheduling that rotates hosts | `SchedulingType.ROUND_ROBIN`; `getLuckyUser`, `isRRWeightsEnabled` |
| **Host** | A team member who can be assigned bookings | `Host` model; `EventType.hosts` `schema.prisma:171` |
| **Managed event type** | A parent event type templating children | `EventType.parentId`/`children` `schema.prisma:190-192` |
| **Idempotency key** | Deterministic key preventing duplicate bookings | `Booking.idempotencyKey` `schema.prisma:855`; extension `booking-idempotency-key.ts` |
| **Credential** | Stored OAuth/API secret for an integration | `Credential` model `schema.prisma:308`; **never expose `.key`** |
| **App / App store** | An integration (calendar, video, payment, CRM) | `packages/app-store/<app>/` |
| **EventManager** | Orchestrates calendar/video create/update/delete | `EventManager.ts:133` |
| **Destination calendar** | Where new events are written | `DestinationCalendar`; `Booking.destinationCalendarId` |
| **Tasker / Task** | The background job queue + a job row | `packages/features/tasker/`; `Task` model `schema.prisma:1825` |
| **Webhook** | A signed HTTP callback on booking events | `Webhook` model `schema.prisma:1142`; `sendPayload.ts` |
| **Trigger event** | A booking lifecycle event that fires webhooks | `WebhookTriggerEvents` enum (e.g. `BOOKING_CREATED`, `BOOKING_CANCELLED`) |
| **Booker** | The public booking UI widget | `packages/features/bookings/Booker/`; `apps/web/modules/bookings/components/Booker.tsx` |
| **Viewer** | The authenticated-user tRPC namespace | `routers/viewer/_router.tsx` |
| **Platform / atoms** | Embeddable React components + platform API | `packages/platform/atoms/`; `apps/api/v2` |
| **PBAC** | Permission-based access control (team roles) | `eventTypes/util.ts` — **currently a stub**, see [risk register](../09-reference/risk-register.md) |
| **OOO** | Out-of-office periods that block availability | `packages/features/ooo/`; `OOORepository` |
| **DI container / token / module** | Dependency-injection wiring primitives | `packages/features/di/` |

## Confusing near-synonyms

- **Slot** vs **Reservation** vs **Booking**: a slot is a *computed possibility*; a reservation is a *soft hold*; a booking is a *committed fact*. Only the booking has correctness guarantees.
- **Availability** (a rule) vs **Busy time** (an occupied interval) vs **Schedule** (a named collection of rules). Slots = schedule expanded minus busy times.
- **Event Type** (the template) vs **Booking** (an instance). One event type → many bookings.
- **Host** vs **Owner** vs **Attendee**: owner created the event type; hosts can be assigned (teams); attendees are the people being scheduled.
- **Task** (tasker job) vs **Trigger** (webhook cause) vs **Webhook** (the subscription). A trigger enqueues a task that sends to a webhook.

Drill: for each of Booking/Slot/Reservation, name the guarantee it does and doesn't provide, and the file that owns it.
