# Data Models

The database schema lives in:

- `packages/prisma/schema.prisma`

Generated Prisma client and helper exports live under:

- `packages/prisma/generated/prisma`
- `packages/prisma/zod`
- `packages/prisma/index.ts`

Do not edit generated output by hand. Change the Prisma schema and create migrations.

## How to read the schema

The schema is large because Cal.diy supports personal scheduling, teams, organizations, app integrations, API/platform access, payments, routing, webhooks, seats, recurring events, and audit behavior.

The core scheduling loop is:

```text
User/Profile/Team
  -> EventType
  -> Schedule/Availability
  -> SelectedCalendar and DestinationCalendar
  -> Booking
  -> Attendee, BookingReference, Payment, BookingSeat
```

## Identity and organization models

### `User`

Represents a person who can log in, create event types, connect calendars, and receive bookings.

Important fields:

- `id`, `uuid`: internal identifiers.
- `username`: public booking URL identifier.
- `name`, `email`: display and login identity.
- `timeZone`, `timeFormat`, `weekStart`: scheduling preferences.
- `locale`: language preference.
- `metadata`: flexible JSON for additional user settings.
- `role`: app-level role.
- `defaultScheduleId`: user's default schedule.
- `identityProvider`: auth provider.
- `twoFactorSecret`, `twoFactorEnabled`, `backupCodes`: two-factor auth data.

Important relations:

- `eventTypes`: event types owned by the user.
- `bookings`: bookings connected to the user.
- `schedules`: availability schedules.
- `availability`: legacy/direct availability records.
- `selectedCalendars`: calendars checked for conflicts.
- `destinationCalendar`: default calendar for writing bookings.
- `credentials`: third-party integration credentials.
- `teams`: memberships.
- `accounts`, `sessions`: auth records.
- `profiles`: org-aware profiles.

When to touch it:

- User profile settings, auth behavior, default schedule behavior, timezone/locale handling, or user-level integration behavior.

### `Profile`

Represents a user's public identity inside an organization context.

Important fields:

- `id`, `uid`
- `userId`
- `organizationId`
- `username`

Important relations:

- Belongs to a `User`.
- Belongs to an organization-like `Team`.
- Can own org-scoped `EventType` records through `profileId`.

Why it exists:

- A user can have different public identity/profile behavior inside an organization.

### `Team`

Represents either a team or an organization.

Important fields:

- `id`
- `name`
- `slug`
- Branding fields
- `isOrganization`
- `metadata`
- booking limit settings

Important relations:

- `members`: `Membership` rows.
- `eventTypes`: team event types.
- `parent`, `children`: organization/team hierarchy.
- `organizationSettings`
- `orgProfiles`
- `platformOAuthClient`

When to touch it:

- Team pages, org behavior, membership permissions, team event types, branding, or organization-level settings.

### `Membership`

Connects a user to a team.

Important fields:

- `teamId`
- `userId`
- `accepted`
- `role`
- `customRole`

Important relation:

- Unique membership for a user/team pair.

When to touch it:

- Invitations, team roles, permission checks, and team membership UI.

## Scheduling configuration models

### `EventType`

Represents a bookable event, such as "30 minute intro call".

Important fields:

- `id`
- `title`
- `slug`
- `description`
- `length`
- `locations`
- `hidden`
- `userId`
- `teamId`
- `profileId`
- `scheduleId`
- `bookingFields`
- `metadata`
- `recurringEvent`
- `periodType`, `periodStartDate`, `periodEndDate`, `periodDays`
- `beforeEventBuffer`, `afterEventBuffer`
- `requiresConfirmation`
- `seatsPerTimeSlot`
- `schedulingType`
- `maxActiveBookingsPerBooker`
- `bookingRequiresAuthentication`

Important relations:

- Belongs to a user, team, or profile depending on event type kind.
- Has many `Booking` records.
- Can have `Availability` records.
- Can have a `Schedule`.
- Can have `SelectedCalendar` overrides.
- Can have a `DestinationCalendar`.
- Can have `Webhook` records.
- Can have `Host` records for team scheduling.
- Can have `HashedLink` private links.
- Can have Cal Video settings.

Important unique rules:

- Slugs are unique per user/team scope, not globally unique.

When to touch it:

- Event type settings, booking forms, durations, locations, team scheduling, buffers, recurring events, seats, confirmation settings, or custom booking fields.

### `Host`

Represents a host assigned to a team event type.

Important fields:

- user/event type relationship fields
- priority or weight style fields
- fixed host behavior
- schedule/location relations

Important relations:

- Connects `User` and `EventType`.
- May reference host locations and schedules.

When to touch it:

- Collective events, round-robin events, fixed hosts, host-specific locations, or host assignment behavior.

### `HostGroup`

Represents a grouping of hosts for event type routing/assignment behavior.

When to touch it:

- Host grouping, routing, and advanced team booking assignment.

### `Schedule`

Represents a named availability schedule.

Important fields:

- `id`
- `userId`
- `name`
- `timeZone`

Important relations:

- Has many `Availability` rows.
- Can be assigned to event types.
- Can be used as a restriction schedule.
- Can be connected to hosts.

When to touch it:

- Availability editor, default schedule behavior, schedule assignment, or timezone schedule rules.

### `Availability`

Represents an availability block.

Important fields:

- `days`: weekdays this block applies to.
- `startTime`, `endTime`: time-of-day boundaries.
- `date`: date-specific availability.
- `userId`
- `eventTypeId`
- `scheduleId`

Important relations:

- Usually belongs to a `Schedule`.
- May relate directly to a user or event type depending on legacy/current usage.

When to touch it:

- Availability editing, schedule calculation, and slot generation.

### `SelectedCalendar`

Represents an external calendar that should be checked for conflicts.

Important fields:

- `userId`
- `integration`
- `externalId`
- `credentialId`
- channel/sync fields
- `eventTypeId`

Important relations:

- Belongs to a user.
- Usually belongs to a `Credential`.
- Can be scoped to an event type.
- Can have cached calendar events.

When to touch it:

- "Check for conflicts" settings, calendar sync, availability calculations, or integration debugging.

### `DestinationCalendar`

Represents the calendar where new booking events should be written.

Important fields:

- `integration`
- `externalId`
- `primaryEmail`
- `userId`
- `eventTypeId`
- `credentialId`
- `customCalendarReminder`

Important relations:

- Can be user-level or event-type-level.
- Uses `Credential` to write to external calendars.

When to touch it:

- "Add bookings to this calendar" behavior, calendar write failures, or per-event destination settings.

## Booking models

### `Booking`

Represents a scheduled meeting/reservation.

Important fields:

- `id`
- `uid`: public unique booking identifier.
- `idempotencyKey`
- `userId`
- `userPrimaryEmail`
- `eventTypeId`
- `title`
- `description`
- `customInputs`
- `responses`
- `startTime`, `endTime`
- `location`
- `status`
- `paid`
- `destinationCalendarId`
- cancellation/rejection/reassignment fields
- `rescheduled`
- `fromReschedule`
- `recurringEventId`
- `smsReminderNumber`
- `metadata`
- `iCalUID`, `iCalSequence`
- `creationSource`

Important relations:

- Belongs to a user/host and often an `EventType`.
- Has many `Attendee` rows.
- Has many `BookingReference` rows.
- Can have `Payment`, `BookingSeat`, tracking, audit, and report records.

When to touch it:

- Booking lifecycle, booking pages, reschedule/cancel behavior, reporting, status transitions, and calendar sync.

### `Attendee`

Represents a guest attending a booking.

Important fields:

- `email`
- `name`
- `timeZone`
- `phoneNumber`
- `locale`
- `bookingId`
- `noShow`

Important relations:

- Belongs to a `Booking`.
- May connect to seat data.

When to touch it:

- Guest information, booking form responses, attendee notifications, and no-show behavior.

### `BookingReference`

Represents an external reference for a booking.

Important fields:

- `type`
- `uid`
- `meetingId`
- `meetingPassword`
- `meetingUrl`
- `externalCalendarId`
- `credentialId`
- `delegationCredentialId`

Important relations:

- Belongs to a `Booking`.
- May reference integration credentials.

When to touch it:

- Calendar event IDs, conferencing links, integration sync, and cancellation/update calls to external providers.

### `Payment`

Represents payment information connected to a booking.

When to touch it:

- Paid bookings, payment redirects, refunds, or payment app integrations.

### `BookingSeat`

Represents a seat in a seated event.

When to touch it:

- Multiple attendees per time slot, seat reservation, seat cancellation, or seated event capacity.

### `BookingAudit`

Represents audit information about booking changes.

When to touch it:

- Compliance, debugging booking lifecycle changes, or adding audit events.

### `BookingReport`

Represents booking reporting metadata.

When to touch it:

- Reporting dashboards, analytics, or internal booking diagnostics.

## Integration models

### `Credential`

Represents encrypted credentials for an external integration.

Important fields:

- `type`
- `key`
- `encryptedKey`
- `userId`
- `teamId`
- `appId`
- `invalid`

Important relations:

- User/team credentials.
- Destination calendars.
- Selected calendars.
- Booking references.
- Delegation credentials.
- Host locations.

When to touch it:

- OAuth integrations, calendar connections, invalid credential behavior, or external provider API calls.

### `App`

Represents app-store integration metadata.

When to touch it:

- Adding/changing app-store integrations or app metadata.

### `Webhook`

Represents an outbound webhook subscription.

Important relations:

- Can connect to users, teams, event types, and platform clients.

When to touch it:

- Webhook delivery, event subscriptions, payload changes, retries, or signing.

## Auth and platform access models

### `Account` and `Session`

NextAuth-related identity records.

When to touch them:

- Auth provider behavior, session persistence, and login debugging.

### `ApiKey`

Represents API keys for external API access.

When to touch it:

- API auth, key creation, revocation, and API v1/v2 access.

### `OAuthClient`, `PlatformOAuthClient`, `AccessToken`, `RefreshToken`

Represent OAuth/platform API clients and issued tokens.

When to touch them:

- Platform API auth, OAuth flows, token rotation, and external developer integrations.

### `VerifiedEmail` and `VerifiedNumber`

Represent verified contact methods.

When to touch them:

- Identity verification, SMS/phone verification, email verification, and trusted attendee/host contact flows.

## Feature flag and entitlement models

### `Feature`, `UserFeatures`, `TeamFeatures`

Represent feature flags or feature access.

When to touch them:

- Feature gating, rollout behavior, or plan/team-specific access.

## Out-of-office and scheduling exception models

### `OutOfOfficeEntry`

Represents periods when a user is unavailable.

When to touch it:

- OOO editor, slot calculation, booking reassignment during OOO, or availability debugging.

## Data-model debugging tips

### Start from the real-world noun

If the user says "my booking did not create a Zoom link", start at `Booking`, then inspect `BookingReference`, `Credential`, and app-store integration code.

If the user says "this time should not be available", start at `EventType`, `Schedule`, `Availability`, `SelectedCalendar`, `Booking`, and `OutOfOfficeEntry`.

### Use schema relations as your map

Prisma relation fields tell you which joins matter. For example, `EventType` links to bookings, hosts, schedules, selected calendars, destination calendar, and webhooks because all of those can affect booking behavior.

### Repositories are preferred entry points

The team prefers repository classes for data access. For event types, start with:

- `packages/features/eventtypes/repositories/eventTypeRepository.ts`

For booking persistence, start with:

- `packages/features/bookings/lib/handleNewBooking/createBooking.ts`

Some older service code still uses Prisma directly. Treat that as existing reality, not necessarily the pattern to copy.
