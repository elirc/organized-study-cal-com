# Practice User Stories

These are fake implementation stories for onboarding. They are written the way a senior engineer might hand work to a junior developer: not just "change X", but "here is how to think through X in this repo".

Do not implement these as part of onboarding. Use them to practice tracing the codebase, identifying ownership boundaries, and planning a safe PR.

## Story 1: Add an internal note field to booking creation

### User story

As an organizer, I want attendees to optionally leave an internal note during booking so that I can see extra context on the booking record without changing the public event description.

### Product scope

Add an optional text field to the public booking form. The value should be saved on the created `Booking` as `internalNote`, which already exists in the Prisma `Booking` model.

Do not add notifications, workflow logic, or calendar description changes in the first version.

### First mental model

This is a booking creation change. Booking creation is special in this repo because the browser does not submit through tRPC. It posts to `/api/book/event`.

Trace:

```text
BookEventForm UI
  -> useHandleBookEvent
  -> booking-to-mutation-input-mapper
  -> useBookings
  -> create-booking.ts
  -> apps/web/pages/api/book/event.ts
  -> RegularBookingService
  -> handleNewBooking/createBooking.ts
  -> Booking.internalNote
```

### Files to inspect first

- `apps/web/modules/bookings/components/BookEventForm/BookEventForm.tsx`
- `apps/web/modules/bookings/components/BookEventForm/BookingFields.tsx`
- `packages/platform/atoms/hooks/bookings/useHandleBookEvent.ts`
- `packages/features/bookings/lib/client/booking-event-form/booking-to-mutation-input-mapper.tsx`
- `packages/features/bookings/lib/bookingCreateBodySchema.ts`
- `apps/web/pages/api/book/event.ts`
- `packages/features/bookings/lib/service/RegularBookingService.ts`
- `packages/features/bookings/lib/handleNewBooking/createBooking.ts`
- `packages/prisma/schema.prisma`

### Implementation plan

1. Confirm the data model already supports the field.

   Search:

   ```powershell
   rg "internalNote" packages/prisma/schema.prisma packages/features apps/web
   ```

   If `Booking.internalNote` exists and is a nullable string, avoid a schema migration.

2. Identify where booking form fields are generated.

   Start in `BookEventForm.tsx` and `BookingFields.tsx`. Determine whether this should be:

   - a hard-coded field always shown on the booking form, or
   - a configurable event type booking field.

   For this story, choose a hard-coded optional field only if product wants it globally. If it should be configurable per event type, this becomes an event type settings story too.

3. Add form state carefully.

   Follow the existing React Hook Form pattern in `BookEventForm.tsx`. Use the existing UI components from `packages/ui`.

   Senior engineer note: do not invent a parallel form state with `useState`. Booking form state already has a form system.

4. Map the field into the booking mutation body.

   Update:

   - `packages/features/bookings/lib/client/booking-event-form/booking-to-mutation-input-mapper.tsx`

   Make sure the mapper includes `internalNote` only when non-empty, or sends `null`/`undefined` consistently with existing body conventions.

5. Update server validation.

   Update:

   - `packages/features/bookings/lib/bookingCreateBodySchema.ts`

   Add `internalNote` as optional string input. Consider trimming and max length. A reasonable first limit would be 1000 or 2000 characters, but match existing booking field limits if one exists.

6. Verify service and persistence support.

   Find where booking data is transformed inside:

   - `RegularBookingService.ts`
   - `handleNewBooking/createBooking.ts`

   Add the field as close as possible to where other booking request fields are converted into Prisma `Booking` data.

   Senior engineer note: avoid passing raw request bodies deep into persistence. Prefer explicitly mapping the field so future readers can see that it is intended.

7. Add tests.

   Good test targets:

   - schema validation for `bookingCreateBodySchema`
   - booking persistence test near `packages/features/bookings/lib/handleNewBooking/createBooking.ts`
   - if there is an existing booking E2E test, add one assertion only if the UI behavior is important

8. Manual verification.

   - Run the app.
   - Open a public booking page.
   - Submit a booking with the note.
   - Inspect `Booking.internalNote` in Prisma Studio.

### Risks and gotchas

- Booking fields and attendee responses are already complex. Do not accidentally store this note inside `responses` unless that is the product decision.
- Booking creation supports recurring, reschedule, dry-run, decoy, seated, and embed flows. Make sure the new field does not break those paths.
- If the note should be organizer-only, do not include it in attendee emails or public confirmation pages.

### Suggested PR shape

- One small UI change.
- One input mapper change.
- One schema validation change.
- One service/persistence mapping change.
- Focused tests.

## Story 2: Add a confirmation toggle to event type creation

### User story

As an organizer, I want to choose "requires confirmation" when creating an event type so that new bookings for that event can start as pending instead of immediately confirmed.

### Product scope

Add a checkbox/toggle to the event type creation dialog. Persist it to `EventType.requiresConfirmation`.

Do not build the full pending-booking management UI in this story.

### First mental model

This is a logged-in event type mutation. Unlike booking creation, event type creation uses tRPC.

Trace:

```text
CreateEventTypeDialog
  -> CreateEventTypeForm
  -> useCreateEventType
  -> trpc.viewer.eventTypesHeavy.create
  -> create.schema.ts
  -> create.handler.ts
  -> EventTypeRepository.create
  -> EventType.requiresConfirmation
```

### Files to inspect first

- `apps/web/modules/event-types/components/CreateEventTypeDialog.tsx`
- `packages/features/eventtypes/components/CreateEventTypeForm.tsx`
- `apps/web/modules/event-types/hooks/useCreateEventType.ts`
- `packages/features/eventtypes/lib/schemas.ts`
- `packages/trpc/server/routers/viewer/eventTypes/heavy/create.schema.ts`
- `packages/trpc/server/routers/viewer/eventTypes/heavy/create.handler.ts`
- `packages/features/eventtypes/repositories/eventTypeRepository.ts`
- `packages/prisma/schema.prisma`

### Implementation plan

1. Confirm the existing data field.

   Search:

   ```powershell
   rg "requiresConfirmation" packages/prisma/schema.prisma packages/features apps/web
   ```

   If `EventType.requiresConfirmation` already exists, no migration is needed.

2. Understand the current create form contract.

   Open:

   - `CreateEventTypeForm.tsx`
   - `useCreateEventType.ts`
   - `createEventTypeInput` in `packages/features/eventtypes/lib/schemas.ts`

   Find the form's default values and submission type.

3. Add UI in the shared feature form.

   Update:

   - `packages/features/eventtypes/components/CreateEventTypeForm.tsx`

   Use the project's existing checkbox/switch component. Do not add a custom control.

   Senior engineer note: creation forms should stay compact. A toggle belongs near other core behavior settings, not in a large explanatory block.

4. Extend validation.

   Update:

   - `packages/features/eventtypes/lib/schemas.ts`

   Add an optional boolean with a default if the schema style supports it.

5. Pass the value through the tRPC layer.

   `create.schema.ts` reuses `createEventTypeInput`, so it may not need a direct change if the shared schema is updated.

   Confirm the router still compiles:

   - `packages/trpc/server/routers/viewer/eventTypes/heavy/_router.ts`

6. Map the field in the handler.

   Update:

   - `packages/trpc/server/routers/viewer/eventTypes/heavy/create.handler.ts`

   Add `requiresConfirmation` when building the Prisma create input.

   Senior engineer note: this handler already handles team/org/default-location behavior. Keep the new field boring and explicit.

7. Confirm repository behavior.

   Check:

   - `packages/features/eventtypes/repositories/eventTypeRepository.ts`

   If it passes through create data generically, no repository change may be needed. If it whitelists fields, add the field there.

8. Add tests.

   Good test target:

   - handler/repository test proving `requiresConfirmation` is persisted
   - schema test if nearby schema tests exist

9. Manual verification.

   - Log in.
   - Create an event type with the toggle enabled.
   - Inspect the event type in Prisma Studio.
   - Create a test booking and confirm existing pending/confirmation behavior uses the saved flag.

### Risks and gotchas

- Team event types may have permission and organization checks. Test personal and team creation if possible.
- The create handler in this Cal.diy codebase has simplified permission behavior. Do not use that as a reason to skip thinking about team authorization.
- Existing edit-event-type UI may already support this field. Keep create and edit behavior consistent.

### Suggested PR shape

- Shared form update.
- Schema update.
- Handler/repository persistence update.
- One focused test.

## Story 3: Show "out of office" blocked reason in slot debugging logs

### User story

As a support engineer, I want slot calculation logs to identify when out-of-office entries remove availability so that I can debug "why is this time unavailable?" faster.

### Product scope

Add developer/support-facing logging only. Do not change the user-facing slot UI.

### First mental model

This is availability computation. Slots are computed by tRPC, not read from a simple table.

Trace:

```text
Booker date/time UI
  -> trpc.viewer.slots.getSchedule
  -> getSchedule.handler.ts
  -> AvailableSlotsService in slots/util.ts
  -> Schedule, Availability, Booking, SelectedCalendar, OutOfOfficeEntry
```

### Files to inspect first

- `packages/trpc/server/routers/viewer/slots/_router.tsx`
- `packages/trpc/server/routers/viewer/slots/getSchedule.handler.ts`
- `packages/trpc/server/routers/viewer/slots/util.ts`
- `packages/features/ooo`
- `packages/prisma/schema.prisma`
- `packages/lib` logging helpers

### Implementation plan

1. Find existing logging conventions.

   Search:

   ```powershell
   rg "logger" packages/trpc/server/routers/viewer/slots packages/features/ooo packages/lib
   rg "NEXT_PUBLIC_LOGGER_LEVEL|LOG_LEVEL|debug" packages/lib apps/web packages/trpc
   ```

   Use the existing logger. Do not add `console.log` in a hot path.

2. Identify where OOO data is loaded.

   In `packages/trpc/server/routers/viewer/slots/util.ts`, locate the `AvailableSlotsService` path that loads out-of-office entries or repositories.

   Senior engineer note: slot calculation is performance-sensitive. Logging must be low-volume and gated by existing debug levels.

3. Decide log shape.

   Good debug log fields:

   - event type id/slug
   - user id or host id
   - requested date range
   - number of OOO entries considered
   - whether OOO affected final slot count

   Avoid logging attendee personal data or raw tokens.

4. Add logging at the right boundary.

   Prefer logging once after OOO entries are applied, not once per slot. Per-slot logs can explode in volume.

5. Add tests only if behavior changes are testable.

   Since this is logging-only, unit tests may not be worth it unless there is an existing logger mock pattern. The better verification may be a focused manual/dev run with debug logging enabled.

6. Manual verification.

   - Create an OOO entry for a seeded user.
   - Open that user's booking page.
   - Request slots covering the OOO period.
   - Confirm logs identify OOO as a filtering factor.

### Risks and gotchas

- Slot generation may be cached through `withSlotsCache`; logs may not appear on cache hits.
- Slot code is shared by public pages and can be high-traffic. Keep logs gated and concise.
- Avoid leaking emails, attendee responses, credential data, or calendar event details.

### Suggested PR shape

- One logging helper or local debug log.
- No product behavior change.
- Manual verification note in PR description.

## Story 4: Add a new webhook event for booking internal-note updates

### User story

As an integration developer, I want to receive a webhook when a booking's internal note changes so that my external system can stay in sync.

### Product scope

Emit a webhook when an existing booking's `internalNote` is updated from the app. This story assumes an internal-note update feature already exists.

Do not build the edit UI in this story. Focus on webhook event modeling and triggering.

### First mental model

This is a side-effect story. The data write may happen in one feature area, but webhook definitions and delivery live elsewhere.

Trace:

```text
booking update handler/service
  -> Booking.internalNote update
  -> webhook trigger/event definition
  -> webhook delivery infrastructure
  -> Webhook model
```

### Files to inspect first

- `packages/features/webhooks`
- `packages/trpc/server/routers/viewer/webhook`
- `packages/prisma/schema.prisma`
- `packages/features/bookings`
- existing booking-created/booking-cancelled webhook trigger code

Useful searches:

```powershell
rg "BOOKING_CREATED|BOOKING_CANCELLED|booking.created|booking.cancelled"
rg "Webhook" packages/features packages/trpc packages/prisma/schema.prisma
rg "internalNote"
```

### Implementation plan

1. Find the existing webhook event taxonomy.

   Search for event constants/enums. Identify naming style:

   - uppercase enum values
   - dot-separated event names
   - payload builders
   - trigger helpers

2. Identify payload shape for booking events.

   Find the payload used for booking created/updated/cancelled. Reuse it if possible.

   Senior engineer note: external webhook payloads are contracts. Avoid adding random one-off shapes.

3. Decide the event name.

   Possible names:

   - `BOOKING_INTERNAL_NOTE_UPDATED`
   - `booking.internal_note_updated`

   Use whatever naming style existing webhook events use.

4. Add event type support.

   This may require updates in:

   - event enum/constants
   - webhook subscription validation
   - webhook UI labels
   - payload typing
   - API docs if webhook events are documented

5. Trigger only after successful persistence.

   The webhook should fire after `Booking.internalNote` is actually updated in the database.

   If the update is inside a service, trigger from the service after the transaction returns. If the update is inside a repository, do not hide side effects in the repository unless existing webhook patterns do that.

6. Include useful payload fields.

   Good fields:

   - booking uid
   - event type id/title
   - organizer id/email if existing payload includes it
   - previous note presence, if allowed
   - new note, only if product/security says webhook consumers may receive it

   Senior engineer note: internal notes may be sensitive. Before including note content in the payload, verify product/security expectations.

7. Add tests.

   Good tests:

   - webhook event can be subscribed to
   - updating note triggers the event
   - no event fires when note is unchanged
   - payload matches existing booking event shape

8. Manual verification.

   - Configure a local webhook receiver.
   - Update a booking internal note.
   - Confirm exactly one webhook is delivered with expected event type.

### Risks and gotchas

- Webhooks are external contracts. Naming and payload shape matter.
- Sensitive internal note content may not be safe to send by default.
- Avoid firing webhooks from failed transactions or dry-run paths.
- Existing webhook retry/delivery infrastructure should be reused.

### Suggested PR shape

- Event constant/schema update.
- Trigger addition in booking update service.
- Payload typing/update.
- Tests around event firing.
- Docs/API reference update if webhook events are documented.

## Story 5: Add a "copy private booking link" action for hashed links

### User story

As an organizer, I want to copy a private booking link for an event type so that I can share a hidden link without making the event publicly discoverable.

### Product scope

Add a UI action on the event type details/settings page to create or copy a hashed link for that event type.

Do not redesign event type sharing. Start with one action that creates/reuses an existing `HashedLink` and copies the URL.

### First mental model

This is an event type management feature with a database-backed link. It likely uses authenticated tRPC because only the organizer should create/copy private links.

Trace:

```text
event type settings UI
  -> authenticated tRPC mutation
  -> event type permission check
  -> HashedLink create/find
  -> return URL
  -> copy to clipboard + toast
```

### Files to inspect first

- `apps/web/modules/event-types`
- `packages/trpc/server/routers/viewer/eventTypes`
- `packages/features/eventtypes`
- `packages/features/hashedLink`
- `packages/prisma/schema.prisma`

Useful searches:

```powershell
rg "HashedLink"
rg "hashedLink"
rg "copy" apps/web/modules/event-types packages/features/eventtypes
rg "navigator.clipboard|clipboard"
```

### Implementation plan

1. Understand the existing hashed link model.

   Inspect `HashedLink` in:

   - `packages/prisma/schema.prisma`

   Identify:

   - relation to `EventType`
   - unique fields
   - link hash/token field
   - whether multiple hashed links per event type are allowed

2. Find existing hashed-link feature code.

   Start with:

   - `packages/features/hashedLink`

   If helpers already exist, reuse them. Do not invent token-generation logic if a project helper exists.

3. Find the event type settings UI location.

   Search in:

   - `apps/web/modules/event-types`

   Identify the page/component where sharing actions live. Add the button near existing link/share controls.

4. Design the API.

   Likely tRPC mutation:

   ```text
   trpc.viewer.eventTypes.createPrivateLink.useMutation()
   ```

   Exact router placement should match existing event type router organization. If heavy mutations live under `eventTypesHeavy`, place it there only if existing patterns suggest it.

5. Add input schema.

   Minimal input:

   - `eventTypeId`

   Use Zod. Validate number/int.

6. Add auth and permission checks.

   Use `authedProcedure`.

   In the handler:

   - load event type by id
   - confirm current user owns it or has team permission
   - handle org/team event types explicitly

   Senior engineer note: this is where junior developers often under-scope. Personal event types are easy; team/org event types are where bugs hide.

7. Create or reuse a hashed link.

   Repository/service behavior should be idempotent:

   - if a hashed link already exists for the event type, return it
   - otherwise create one

   Prefer a repository method over direct Prisma in the handler if local patterns support it.

8. Return a full URL or token?

   Prefer returning enough data for the frontend to build the URL using existing URL helpers/constants.

   Check existing usage of:

   - `NEXT_PUBLIC_WEBAPP_URL`
   - link-building helpers in `packages/lib`

   Senior engineer note: avoid hardcoding `localhost:3000` or string-concatenating URLs in a component.

9. Add UI behavior.

   In the event type module:

   - call mutation
   - copy returned link with existing clipboard helper/pattern
   - show success/error toast with Sonner or existing toast utility
   - handle loading/disabled state

10. Add tests.

   Good tests:

   - unauthorized user cannot create link
   - owner can create link
   - second call reuses existing link
   - team permission behavior if team event types are supported

11. Manual verification.

   - Log in as seeded user.
   - Create/open an event type.
   - Click copy private link.
   - Paste link in a private browser window.
   - Confirm booking page loads even if event type is hidden, if that is existing hashed-link behavior.

### Risks and gotchas

- Link creation must be permission-protected.
- Existing hidden/private-link behavior may already have assumptions. Trace current hashed link usage before changing routes.
- Team/org event type permissions are more complex than personal event types.
- Clipboard APIs can fail outside secure contexts; handle failure with a visible toast.

### Suggested PR shape

- tRPC schema/router/handler.
- Feature repository/helper reuse or addition.
- One event type UI action.
- Tests for auth and idempotency.
- No broad sharing-page redesign.

## Story 6: Add helper text to custom booking fields

### User story

As an organizer, I want to add helper text to custom booking fields so that attendees understand what information I am asking for.

### Product scope

Allow event type booking fields to store and render optional helper text. Do not redesign the booking field builder.

### First mental model

This touches event type configuration and public booking rendering. The data probably lives inside `EventType.bookingFields`, which is JSON, so the main work is schema/type/UI mapping rather than a new Prisma column.

### Files to inspect first

- `packages/prisma/schema.prisma`
- `packages/features/eventtypes`
- `packages/features/bookings/lib/client/booking-event-form`
- `apps/web/modules/bookings/components/BookEventForm/BookingFields.tsx`
- event type settings components under `apps/web/modules/event-types`

### Implementation plan

1. Inspect the current `bookingFields` shape with `rg "bookingFields" packages/features apps/web`.
2. Find the form builder/editor that lets organizers configure custom booking fields.
3. Extend the field schema/type with optional `helperText`.
4. Add an input in the event type settings UI using existing form patterns.
5. Render helper text in `BookingFields.tsx` under the corresponding field.
6. Ensure helper text is not treated as attendee response data.
7. Add tests around schema parsing and rendering if nearby tests exist.
8. Manually create an event type with helper text and confirm it appears on the public booker.

### Risks and gotchas

- `bookingFields` is JSON, so type drift is easy. Update shared types/schemas, not just one component.
- Existing fields may not have `helperText`; keep backward compatibility.
- Avoid rendering raw HTML unless the existing field system explicitly supports rich text.

### Suggested PR shape

- Shared booking field type/schema update.
- Event type settings UI update.
- Booker rendering update.
- Focused schema/rendering tests.

## Story 7: Add default before/after buffers for newly created event types

### User story

As an organizer, I want new event types to start with default buffer times so that I have breathing room between meetings.

### Product scope

Set default `beforeEventBuffer` and `afterEventBuffer` values during event type creation. Do not change existing event types.

### First mental model

This is a creation-default story. The right place is event type creation form defaults and/or the server create handler, not a migration.

### Files to inspect first

- `packages/features/eventtypes/components/CreateEventTypeForm.tsx`
- `apps/web/modules/event-types/hooks/useCreateEventType.ts`
- `packages/features/eventtypes/lib/schemas.ts`
- `packages/trpc/server/routers/viewer/eventTypes/heavy/create.handler.ts`
- `packages/features/eventtypes/repositories/eventTypeRepository.ts`
- `packages/prisma/schema.prisma`

### Implementation plan

1. Confirm `EventType.beforeEventBuffer` and `EventType.afterEventBuffer` exist.
2. Search current default behavior with `rg "beforeEventBuffer|afterEventBuffer"`.
3. Decide whether defaults are frontend-only, server-only, or both.
4. Prefer server defaults in `create.handler.ts` so API callers get consistent behavior.
5. Optionally show the defaults in `CreateEventTypeForm.tsx` if fields are exposed there.
6. Add schema defaults or explicit fallback logic.
7. Test creating an event type without buffer input and confirm saved values.

### Risks and gotchas

- Buffers affect slot availability, so verify booking slots after creation.
- Team-managed event types may inherit or override settings differently.
- Do not change update/edit behavior unless product asks.

### Suggested PR shape

- Creation default logic.
- One handler/repository test.
- Manual verification with public slots.

## Story 8: Show the organizer timezone on public booking pages

### User story

As an attendee, I want to see the organizer's timezone on the booking page so that I understand how the meeting time relates to their working hours.

### Product scope

Display organizer timezone in the public event details area. Do not change slot calculation or timezone selection.

### First mental model

This is a read-only public booking page change. Start in the booking page server loader, then the public view and Booker UI.

### Files to inspect first

- `apps/web/app/(booking-page-wrapper)/[user]/[type]/page.tsx`
- `apps/web/server/lib/[user]/[type]/getServerSideProps.ts`
- `apps/web/modules/users/views/users-type-public-view.tsx`
- `apps/web/modules/bookings/components/Booker.tsx`
- `packages/features/eventtypes`

### Implementation plan

1. Inspect the props returned by `getServerSideProps.ts`.
2. Confirm whether organizer/user `timeZone` is already included in public event data.
3. If present, pass it through to the display component.
4. If missing, update the event repository selection carefully to include only safe public data.
5. Render timezone near event title/duration/location.
6. Use existing date/time formatting helpers; do not invent timezone labels.
7. Manually check single-user, team, and dynamic booking pages.

### Risks and gotchas

- Team events may have multiple hosts with different timezones.
- Dynamic/group bookings may not have one obvious organizer timezone.
- Avoid exposing private user data beyond timezone.

### Suggested PR shape

- Public data selection update if needed.
- Small UI display change.
- Fallback behavior for team/dynamic events.

## Story 9: Require cancellation reason for organizer cancellations

### User story

As an attendee, I want organizer cancellations to include a reason so that I understand why my meeting was cancelled.

### Product scope

Require a cancellation reason only when the organizer cancels a booking from the app. Do not require attendees to provide a reason.

### First mental model

This is a booking lifecycle change. Find cancellation UI, mutation/API path, validation, service logic, and `Booking` cancellation fields.

### Files to inspect first

- `apps/web/modules/bookings`
- `packages/trpc/server/routers/viewer/bookings`
- `packages/features/bookings`
- `packages/prisma/schema.prisma`
- `packages/emails`

### Implementation plan

1. Search `rg "cancellationReason|cancelBooking|cancelledBy|cancellation"`.
2. Identify organizer cancellation UI and the mutation/API endpoint it calls.
3. Confirm existing `Booking` fields for cancellation reason and actor.
4. Add frontend validation in the organizer cancellation form.
5. Add server validation in the tRPC schema or API body schema.
6. Ensure attendee cancellation path remains optional if product wants that distinction.
7. Confirm cancellation emails already include reason; if not, update templates only if in scope.
8. Test organizer cancellation without reason, with reason, and attendee cancellation.

### Risks and gotchas

- Cancellation may be triggered by API, workflows, OOO, or reassignment logic. Scope only organizer app cancellation unless product broadens it.
- Emails and webhooks may expose the reason; confirm expected behavior.
- Reschedules can involve cancellation-like state transitions.

### Suggested PR shape

- UI validation.
- Server schema validation.
- Service-level guard.
- Tests for organizer vs attendee behavior.

## Story 10: Add a "resend confirmation email" action for bookings

### User story

As an organizer, I want to resend a booking confirmation email so that I can help attendees who lost the original message.

### Product scope

Add an organizer-only action on booking details to resend the existing confirmation email. Do not create a new email template.

### First mental model

This is an authenticated booking action with a side effect. It should use existing email services/templates and permission checks.

### Files to inspect first

- `apps/web/modules/bookings`
- `packages/trpc/server/routers/viewer/bookings`
- `packages/features/bookings`
- `packages/emails`
- `packages/prisma/schema.prisma`

### Implementation plan

1. Find booking detail actions with `rg "resend|confirmation|BookingDetails" apps/web packages/features`.
2. Find existing confirmation email send code in booking creation.
3. Add an authenticated tRPC mutation for resend.
4. Input should be minimal: booking uid or id.
5. Handler must verify current user can manage the booking.
6. Reuse existing email builder/send function.
7. Add rate limiting if there is an existing pattern for email resend actions.
8. Add a UI action with loading state and toast.
9. Test permission failure, successful send, and missing booking.

### Risks and gotchas

- Do not send to the wrong attendee list; use persisted booking attendees.
- Avoid duplicating booking creation side effects like webhooks/calendar writes.
- Rate limiting matters because this sends email.

### Suggested PR shape

- tRPC mutation.
- Email service reuse.
- One booking detail action.
- Permission and success tests.

## Story 11: Show invalid calendar credential warnings in settings

### User story

As an organizer, I want to see when a connected calendar credential is invalid so that I can reconnect it before bookings fail.

### Product scope

Display a warning in calendar settings for credentials where `Credential.invalid` is true.

### First mental model

This is settings UI reading integration credential state. It likely uses tRPC and existing calendar/credential routers.

### Files to inspect first

- `apps/web/modules/settings`
- `packages/trpc/server/routers/viewer/credentials`
- `packages/trpc/server/routers/viewer/calendars`
- `packages/features/credentials`
- `packages/prisma/schema.prisma`

### Implementation plan

1. Search for connected calendar settings with `rg "SelectedCalendar|DestinationCalendar|Credential.invalid|connected calendars"`.
2. Find the tRPC query that loads credentials/calendars.
3. Confirm `invalid` is selected in the server query.
4. If missing, add `invalid` to the repository/query selection.
5. Render a warning badge/banner next to affected credentials.
6. Link to existing reconnect flow if one exists.
7. Test with a local credential marked invalid in Prisma Studio.

### Risks and gotchas

- Credential keys are sensitive. Only expose safe metadata and the invalid boolean.
- Some integrations may not support reconnect the same way.
- Settings pages may be shared between user and team credentials.

### Suggested PR shape

- Query selection update.
- Settings UI badge/banner.
- Manual verification with invalid credential.

## Story 12: Add duration preset buttons to event type creation

### User story

As an organizer, I want quick duration presets when creating an event type so that I can choose common meeting lengths faster.

### Product scope

Add buttons for common durations such as 15, 30, 45, and 60 minutes. Keep the existing custom duration input.

### First mental model

This is a UI-only improvement unless validation currently rejects a preset. Event type creation already supports `length`.

### Files to inspect first

- `packages/features/eventtypes/components/CreateEventTypeForm.tsx`
- `packages/features/eventtypes/lib/schemas.ts`
- `apps/web/modules/event-types/hooks/useCreateEventType.ts`

### Implementation plan

1. Inspect how `length` is registered in React Hook Form.
2. Confirm min/max constants: `MIN_EVENT_DURATION_MINUTES` and `MAX_EVENT_DURATION_MINUTES`.
3. Add preset buttons that call the form setter for `length`.
4. Keep the custom numeric input.
5. Ensure selected preset visual state follows current form value.
6. Verify keyboard/accessibility behavior with existing button components.
7. Test creation with preset and custom values.

### Risks and gotchas

- Do not hard-code values outside schema bounds.
- Avoid local state that can drift from form state.
- Mobile layout should not crowd the create dialog.

### Suggested PR shape

- One form UI change.
- No backend change unless schema needs alignment.
- Small component test if the form has existing tests.

## Story 13: Add API v2 endpoint to list current user's event types

### User story

As an API consumer, I want an API v2 endpoint to list my event types so that I can build external scheduling dashboards.

### Product scope

Add a read-only API v2 endpoint for authenticated users. Return a minimal list of event type ids, titles, slugs, lengths, and hidden status.

### First mental model

This belongs in `apps/api/v2`, not the web tRPC layer.

### Files to inspect first

- `apps/api/v2`
- API v2 auth guards/modules/controllers
- `packages/features/eventtypes/repositories/eventTypeRepository.ts`
- `packages/prisma/schema.prisma`

### Implementation plan

1. Inspect API v2 module/controller patterns.
2. Find an existing "current user" authenticated endpoint.
3. Add DTO/response type following Nest conventions.
4. Add controller route, for example `GET /v2/me/event-types`, matching existing routing style.
5. Use an existing event type repository or add a read method.
6. Select only safe fields.
7. Add Swagger decorators/OpenAPI metadata.
8. Add Jest tests for auth required and successful response.

### Risks and gotchas

- Do not reuse web tRPC auth assumptions in API v2.
- Public API responses are contracts; keep them stable and documented.
- Team event types may need explicit inclusion/exclusion rules.

### Suggested PR shape

- API v2 controller/DTO/service update.
- Repository read method if needed.
- Swagger docs.
- API tests.

## Story 14: Add filtering to the admin watchlist page

### User story

As an admin, I want to filter watchlist entries by type/status so that I can review suspicious activity faster.

### Product scope

Add UI filters to the existing admin watchlist view. Do not change watchlist detection logic.

### First mental model

This is admin UI plus tRPC query input. Look for `watchlist` under admin routers and feature folders.

### Files to inspect first

- `packages/trpc/server/routers/viewer/admin/watchlist`
- `packages/features/watchlist`
- `apps/web/modules/settings/admin` or admin modules
- `packages/prisma/schema.prisma`

### Implementation plan

1. Search `rg "watchlist" apps/web packages/trpc packages/features`.
2. Identify current query procedure and input schema.
3. Add optional filter fields to the schema.
4. Apply filters in repository/query layer.
5. Add filter controls using existing select/menu components.
6. Keep filter state in URL query params if admin pages already do this.
7. Test query filtering at the handler/repository level.

### Risks and gotchas

- Admin endpoints must remain auth/permission protected.
- Filtering should not accidentally broaden access to entries.
- Use indexes if filtering fields are large/high-volume and already indexed.

### Suggested PR shape

- tRPC schema/query update.
- Admin UI controls.
- Handler/repository tests.

## Story 15: Add a "copy schedule" action

### User story

As an organizer, I want to duplicate an existing schedule so that I can create variations without rebuilding availability from scratch.

### Product scope

Add an authenticated action that copies a `Schedule` and its `Availability` rows for the current user.

### First mental model

This is a schedule mutation. It should copy relational data carefully in a transaction.

### Files to inspect first

- `packages/features/schedules`
- `packages/trpc/server/routers/viewer/availability`
- `packages/trpc/server/routers/viewer/availability/schedule`
- `apps/web/modules/availability` or schedule settings modules
- `packages/prisma/schema.prisma`

### Implementation plan

1. Search `rg "schedule.*create|Availability" packages/trpc/server/routers/viewer packages/features/schedules apps/web`.
2. Find existing create/update schedule procedures.
3. Add `duplicate` tRPC mutation with `scheduleId` and optional new name.
4. Verify current user owns the source schedule.
5. In service/repository, create the new schedule and copied availability rows in a transaction.
6. Avoid copying event type assignments unless product explicitly wants that.
7. Add UI action in schedule list/detail menu.
8. Test ownership, copied availability count, and no copied event type assignments.

### Risks and gotchas

- Timezone should copy from source schedule.
- Availability rows may be date-specific or weekly; copy both.
- Default schedule behavior should not change.

### Suggested PR shape

- tRPC mutation.
- Transactional copy service/repository.
- Schedule UI menu action.
- Ownership and persistence tests.

## Story 16: Add max recurring booking occurrences validation

### User story

As an organizer, I want recurring bookings to have a maximum occurrence limit so that attendees cannot accidentally create too many meetings.

### Product scope

Add server-side validation for recurring booking occurrence count. Use an environment/default constant unless event-type-specific settings already exist.

### First mental model

This is booking creation validation. Recurring booking input is mapped on the client, then validated and processed by booking service code.

### Files to inspect first

- `packages/features/bookings/lib/client/booking-event-form/booking-to-mutation-input-mapper.tsx`
- `packages/features/bookings/lib/bookingCreateBodySchema.ts`
- `apps/web/modules/bookings/hooks/useBookings.ts`
- `packages/features/bookings/lib/service/RegularBookingService.ts`

### Implementation plan

1. Search `rg "recurring" packages/features/bookings apps/web/modules/bookings`.
2. Find where recurring booking arrays are built and submitted.
3. Add or confirm schema validation for max array length.
4. Add service-level guard in `RegularBookingService` so non-web callers are protected too.
5. Return a clear error message/code.
6. Update frontend mutation error handling if needed.
7. Add tests for under-limit, at-limit, and over-limit submissions.

### Risks and gotchas

- Recurring bookings may be submitted as multiple booking objects.
- API callers and embeds should get the same protection.
- Do not rely only on client-side checks.

### Suggested PR shape

- Schema validation.
- Service guard.
- Error handling.
- Focused booking tests.

## Story 17: Validate success redirect URLs for event types

### User story

As an administrator, I want event type success redirect URLs to be validated so that organizers cannot accidentally configure unsafe redirects.

### Product scope

Validate `EventType.successRedirectUrl` on create/update. Allow only valid `http` or `https` URLs, and consider whether same-origin or allowlisted domains are required.

### First mental model

This is event type settings validation with security impact.

### Files to inspect first

- `packages/features/eventtypes/lib/schemas.ts`
- `packages/trpc/server/routers/viewer/eventTypes`
- `packages/features/eventtypes/repositories/eventTypeRepository.ts`
- `packages/prisma/schema.prisma`
- URL safety helpers in `packages/lib`

### Implementation plan

1. Search `rg "successRedirectUrl|getSafeRedirectUrl|redirect" packages/features apps/web packages/lib`.
2. Find create and update schemas for event types.
3. Use an existing safe URL helper if available.
4. Add validation to schemas and/or handlers.
5. Confirm client forms show validation errors cleanly.
6. Test invalid protocol, malformed URL, valid URL, and empty URL.

### Risks and gotchas

- Redirect behavior is security-sensitive.
- Existing event types may already have values; avoid breaking reads.
- Create and update paths must stay consistent.

### Suggested PR shape

- Shared validation helper if needed.
- Event type schema updates.
- Tests for allowed/rejected URLs.

## Story 18: Add attendee locale to booking webhook payloads

### User story

As an integration developer, I want booking webhook payloads to include attendee locale so that I can localize downstream messages.

### Product scope

Add attendee locale to existing booking webhook payloads when available.

### First mental model

This is webhook payload shape, not booking creation behavior. The data already exists on `Attendee.locale`.

### Files to inspect first

- `packages/features/webhooks`
- webhook payload builders
- `packages/features/bookings`
- `packages/prisma/schema.prisma`

### Implementation plan

1. Search `rg "Attendee.*locale|locale" packages/features/webhooks packages/features/bookings`.
2. Find booking webhook payload builder.
3. Confirm attendee locale is selected when booking data is loaded.
4. Add locale field to payload type/schema.
5. Update tests/snapshots for booking webhook payloads.
6. Update docs/API reference if webhook payloads are documented.

### Risks and gotchas

- Webhook payloads are contracts; adding fields is usually safe, renaming is not.
- Bookings may have multiple attendees with different locales.
- Locale can be null/undefined; keep payload backward-compatible.

### Suggested PR shape

- Payload selection/type update.
- Webhook tests.
- Docs update if applicable.

## Story 19: Add visible retry status for webhook deliveries

### User story

As an organizer, I want to see whether a webhook delivery is retrying so that I know if an integration failure is temporary.

### Product scope

Display retry status on an existing webhook delivery/log UI. Do not change retry scheduling.

### First mental model

This is read-only UI over webhook delivery data. Find existing webhook logs before changing backend behavior.

### Files to inspect first

- `apps/web/modules/webhooks` or webhook settings modules
- `packages/features/webhooks`
- `packages/trpc/server/routers/viewer/webhook`
- `packages/prisma/schema.prisma`

### Implementation plan

1. Search `rg "webhook.*log|delivery|retry" apps/web packages/features packages/trpc`.
2. Identify model/fields that track attempts or retry state.
3. Ensure the tRPC query selects retry-related fields.
4. Add UI badge/status text using existing status components.
5. Add empty/unknown fallback for old rows.
6. Test rendering for success, failed, retrying, and exhausted states if fixtures exist.

### Risks and gotchas

- Delivery log models may not store an explicit retry state; you may need to derive it from attempts/status/next retry time.
- Avoid changing retry behavior in a display-only story.
- Do not expose secret webhook headers/payload data in UI.

### Suggested PR shape

- Query selection update if needed.
- UI status display.
- Rendering tests or manual verification.

## Story 20: Add booking title max-length validation

### User story

As an organizer, I want booking titles to have a reasonable maximum length so that calendar events and emails stay readable.

### Product scope

Validate booking title length during booking creation. Show a useful client/server error.

### First mental model

Booking title may be generated from event type and attendee data, not directly typed by the attendee. Validate at the booking service boundary.

### Files to inspect first

- `packages/features/bookings/lib/bookingCreateBodySchema.ts`
- `packages/features/bookings/lib/service/RegularBookingService.ts`
- `packages/features/bookings/lib/handleNewBooking/createBooking.ts`
- `packages/features/CalendarEventBuilder.ts`

### Implementation plan

1. Search `rg "title" packages/features/bookings packages/features/CalendarEventBuilder.ts`.
2. Identify where final booking title is produced.
3. Define a max length constant near existing booking constants.
4. Validate generated title before persistence and before external calendar creation.
5. Decide whether to reject or truncate. Prefer reject unless product wants truncation.
6. Add tests for title at limit and over limit.
7. Confirm calendar event builder handles the same title.

### Risks and gotchas

- Different calendar providers may have different practical limits.
- Generated titles can include names, event titles, or team data.
- Truncation can hide important information; rejection needs clear messaging.

### Suggested PR shape

- Constant + validation.
- Booking service test.
- Error handling update if needed.

## Story 21: Add no-show reason when marking attendee no-show

### User story

As an organizer, I want to record why an attendee was marked no-show so that later reporting has context.

### Product scope

Add optional no-show reason to the mark-no-show flow. This may require a schema migration if no field exists.

### First mental model

This is a booking lifecycle/reporting change. It might touch `Attendee`, no-show feature code, and reporting.

### Files to inspect first

- `packages/features/handleMarkNoShow.ts`
- `packages/features/handleMarkNoShow.test.ts`
- `packages/features/noShow`
- `packages/prisma/schema.prisma`
- booking detail UI under `apps/web/modules/bookings`

### Implementation plan

1. Search `rg "noShow|markNoShow|no-show"`.
2. Confirm whether a reason field already exists on `Attendee`, `Booking`, or report models.
3. If no field exists, add Prisma schema field and migration.
4. Add optional reason to the UI action/modal.
5. Add schema validation for max length.
6. Persist reason in the no-show handler.
7. Update reporting views only if already showing no-show details.
8. Add tests around handler persistence.

### Risks and gotchas

- Schema migration adds more review weight.
- No-show can be host-level or attendee-level; store reason on the right model.
- Reasons may contain sensitive text.

### Suggested PR shape

- Migration if needed.
- UI input.
- Handler/schema update.
- Persistence tests.

## Story 22: Add org slug to public profile metadata

### User story

As a developer working on organization booking pages, I want public profile data to include the organization slug so that links can be generated consistently.

### Product scope

Expose organization slug only where public profile data already includes organization context. Do not expose private team settings.

### First mental model

This is public data loading around `Profile`, `Team`, and booking page routing.

### Files to inspect first

- `apps/web/server/lib/[user]/[type]/getServerSideProps.ts`
- `packages/features/profile`
- `packages/prisma/schema.prisma`
- repositories used by public event/profile loading

### Implementation plan

1. Search `rg "Profile|organizationId|orgSlug|organization.slug" apps/web/server packages/features`.
2. Identify public profile DTO/data shape.
3. Add organization slug to the repository selection only when organization data is already public.
4. Update TypeScript types.
5. Use the field in link generation only if needed by the consumer.
6. Test personal profile and org profile cases.

### Risks and gotchas

- Public data exposure must be intentional.
- Personal profiles may not have an organization.
- Organization/team models share `Team`; avoid confusing team slug and organization slug.

### Suggested PR shape

- Repository selection/type update.
- Consumer update.
- Public/private data check in tests.

## Story 23: Add default phone country selection on booking forms

### User story

As an attendee, I want the phone number field to default to my locale's country so that entering my number is easier.

### Product scope

When a booking form includes a phone field, default country selection based on locale or timezone if available. Do not change phone validation rules.

### First mental model

This is booking form UI behavior. The submitted value should remain compatible with existing booking response handling.

### Files to inspect first

- `apps/web/modules/bookings/components/BookEventForm/BookingFields.tsx`
- phone input components in `packages/ui`
- i18n/locale helpers in `packages/i18n`
- booking input mapper

### Implementation plan

1. Search `rg "phoneNumber|PhoneInput|phone" apps/web packages/ui packages/features/bookings`.
2. Identify the phone field component and its props.
3. Find existing locale from booker props/session/browser.
4. Map locale to country using an existing helper/library if available.
5. Pass default country to phone input.
6. Keep user override behavior intact.
7. Test locale with known country and fallback locale.

### Risks and gotchas

- Locale is not always country, for example `en`.
- Timezone-to-country mapping can be ambiguous.
- Do not alter stored phone number format unexpectedly.

### Suggested PR shape

- Small phone field prop update.
- Helper reuse or tiny mapping helper.
- UI/manual verification.

## Story 24: Add integration category badges in app-store settings

### User story

As an organizer, I want app integrations to show category badges so that I can quickly understand whether an app is for calendars, video, CRM, payments, or analytics.

### Product scope

Display existing app category metadata in app-store/settings UI. Do not change app install behavior.

### First mental model

This is app-store metadata display. The source of truth is likely under `packages/app-store`.

### Files to inspect first

- `packages/app-store`
- `apps/web/modules/apps` or app-store settings modules
- `packages/trpc/server/routers/viewer/apps`
- app metadata/types in `packages/types` or `packages/app-store`

### Implementation plan

1. Search `rg "category|categories|app-store|installed apps" packages/app-store apps/web packages/trpc`.
2. Confirm whether app category already exists in metadata.
3. Ensure app listing query returns category data.
4. Add badges using existing badge/chip UI components.
5. Define fallback for uncategorized apps.
6. Manually verify calendar, video, payment, and CRM apps.

### Risks and gotchas

- Many app-store packages exist; metadata may be inconsistent.
- Avoid changing install/config schemas.
- Badge labels should come from existing category names or translation keys.

### Suggested PR shape

- Query/data plumbing if needed.
- App listing UI badge.
- No install flow changes.

## Story 25: Add email template preview for booking confirmation

### User story

As a developer, I want to preview the booking confirmation email locally so that I can safely change email copy and layout.

### Product scope

Add a local/dev-only preview route or script for booking confirmation emails. Do not expose it in production.

### First mental model

This is developer tooling around `packages/emails`, not booking creation behavior.

### Files to inspect first

- `packages/emails`
- `apps/web/pages/api` or dev-only preview routes
- existing email preview/storybook tooling
- `packages/features/bookings` email send code

### Implementation plan

1. Search `rg "preview|email preview|Booking.*Email|confirmation" packages/emails apps/web`.
2. Find the booking confirmation email component/template.
3. Check if a preview mechanism already exists.
4. Add a dev-only preview route or script using realistic fake booking data.
5. Gate it behind `NODE_ENV !== "production"` or existing dev-only guard.
6. Include locale/timezone variants if easy.
7. Document how to run it in `docs/onboarding/07-local-dev.md` or package README if accepted.

### Risks and gotchas

- Never expose fake/dev preview endpoints in production.
- Email templates may require many booking fields; use a realistic fixture.
- Avoid triggering real email sends from preview.

### Suggested PR shape

- Preview fixture.
- Dev-only preview route/script.
- Documentation note.

## Story 26: Add "hide weekends" preference to the booker

### User story

As an attendee, I want the booking calendar to hide weekends when the organizer never offers weekend slots so that the calendar is easier to scan.

### Product scope

Hide weekends only when the loaded event/schedule has no weekend availability. Do not add a user setting in the first version.

### First mental model

This is booker UI derived from availability data. Slot calculation remains unchanged.

### Files to inspect first

- `apps/web/modules/bookings/components/Booker.tsx`
- date picker components used by the Booker
- `packages/trpc/server/routers/viewer/slots/getSchedule.handler.ts`
- `packages/trpc/server/routers/viewer/slots/util.ts`

### Implementation plan

1. Identify the DatePicker component and props.
2. Determine whether event availability includes weekdays in initial props or slot response.
3. Derive `shouldHideWeekends` only when Saturday/Sunday are never available.
4. Pass the flag into DatePicker using existing props or add a small prop.
5. Confirm mobile and desktop layouts.
6. Test event types with weekday-only and weekend availability.

### Risks and gotchas

- Date-specific weekend availability should prevent hiding weekends.
- Dynamic/team events may combine hosts with different schedules.
- Do not hide weekends if slots exist due to overrides.

### Suggested PR shape

- Derived UI flag.
- DatePicker prop/update.
- Manual verification across event types.

## Story 27: Add selected-calendar count to event type settings

### User story

As an organizer, I want to see how many calendars are checked for conflicts for an event type so that I can catch misconfiguration quickly.

### Product scope

Display a count of selected calendars in event type settings. Do not change selected-calendar management.

### First mental model

This is a read-only event type settings display backed by `SelectedCalendar`.

### Files to inspect first

- `apps/web/modules/event-types`
- `packages/trpc/server/routers/viewer/calendars`
- `packages/features/selectedCalendar`
- `packages/prisma/schema.prisma`

### Implementation plan

1. Search `rg "SelectedCalendar|selectedCalendars" apps/web packages/trpc packages/features`.
2. Find how event type settings load calendars.
3. If count is already in data, render it.
4. If not, add a count query or include selected calendars safely.
5. Use singular/plural label through i18n if surrounding UI does.
6. Test event type with zero, one, and multiple selected calendars.

### Risks and gotchas

- User-level selected calendars and event-type-specific selected calendars may differ.
- Avoid loading full credential data just to count calendars.
- Team event types may use host calendars.

### Suggested PR shape

- Query selection/count update.
- Small settings UI display.
- No mutation changes.

## Story 28: Add audit entry when event type location changes

### User story

As an admin, I want event type location changes to be auditable so that I can investigate unexpected conferencing or in-person location changes.

### Product scope

Record an audit entry when `EventType.locations` changes. Do not build a new audit UI.

### First mental model

This is an event type update side effect. Find existing audit patterns before creating a new model.

### Files to inspect first

- `packages/features/eventtypes`
- `packages/trpc/server/routers/viewer/eventTypes`
- audit packages/models such as `booking-audit`
- `packages/prisma/schema.prisma`

### Implementation plan

1. Search `rg "Audit|audit|locations" packages/features packages/trpc packages/prisma/schema.prisma`.
2. Identify whether there is a generic audit mechanism or only booking-specific audit.
3. If no event type audit model exists, write a design note before adding schema.
4. Detect location changes in the event type update service/handler after loading previous value.
5. Record actor, event type id, previous value, new value, and timestamp.
6. Avoid logging secrets from location metadata.
7. Add tests for changed vs unchanged locations.

### Risks and gotchas

- Some location objects may include provider metadata.
- JSON diffing can be noisy if order changes.
- Adding a new audit table is larger than using an existing audit system.

### Suggested PR shape

- Audit mechanism decision.
- Update handler/service change detection.
- Persistence test.
- No UI work.

## Story 29: Add "booking source" filter to bookings list

### User story

As an organizer, I want to filter bookings by creation source so that I can distinguish web app bookings from API-created bookings.

### Product scope

Add a filter to the bookings list using `Booking.creationSource`. Do not change creation source assignment.

### First mental model

This is bookings list query + UI filtering. The field already exists on `Booking`.

### Files to inspect first

- `apps/web/modules/bookings`
- `packages/trpc/server/routers/viewer/bookings`
- `packages/features/bookings`
- `packages/prisma/schema.prisma`

### Implementation plan

1. Search `rg "creationSource|bookings list|getBookings"`.
2. Identify bookings list query input schema.
3. Add optional `creationSource` filter.
4. Apply filter in repository/query layer.
5. Add UI select using existing filter controls.
6. Preserve URL state if the bookings list uses query params.
7. Test query filtering and UI selection.

### Risks and gotchas

- Older bookings may have null/unknown creation source.
- Filter labels should be user-friendly, not raw enum names.
- Do not change booking creation paths.

### Suggested PR shape

- tRPC schema/query update.
- UI filter.
- Tests for filtering.

## Story 30: Add lightweight app health check for web

### User story

As an operator, I want a lightweight health check endpoint for the web app so that deployment infrastructure can verify the app is alive.

### Product scope

Add a simple web health endpoint that returns status and timestamp. Do not perform expensive database or integration checks.

### First mental model

This is a Next API route or App Router route handler. It should be intentionally boring and fast.

### Files to inspect first

- `apps/web/app/api`
- `apps/web/pages/api`
- existing health/status endpoints
- deployment config references

### Implementation plan

1. Search `rg "health|ready|status" apps/web apps/api`.
2. Choose route style consistent with existing web API endpoints.
3. Add endpoint such as `/api/health`.
4. Return JSON with status, app name, and timestamp.
5. Avoid DB checks unless product specifically asks for readiness, not liveness.
6. Add a tiny route test if web API route tests exist.
7. Document endpoint for operators.

### Risks and gotchas

- Liveness and readiness are different. Do not claim DB readiness if you do not check DB.
- Avoid exposing env secrets or version internals.
- Make sure the route is public if infrastructure needs unauthenticated access.

### Suggested PR shape

- One API route.
- Optional small test.
- Ops/docs note.

## Story 31: Add event type search by slug in dashboard

### User story

As an organizer with many event types, I want to search by slug so that I can find a booking link quickly.

### Product scope

Extend existing event type search/filter to include slug. Do not rebuild event type navigation.

### First mental model

This is event type list query/filter behavior plus UI search.

### Files to inspect first

- `apps/web/modules/event-types`
- `packages/trpc/server/routers/viewer/eventTypes`
- `packages/features/eventtypes/repositories/eventTypeRepository.ts`

### Implementation plan

1. Search `rg "getEventTypesFromGroup|event type search|query" apps/web packages/trpc/server/routers/viewer/eventTypes`.
2. Identify whether search is client-side or server-side.
3. If client-side, include slug in filter predicate.
4. If server-side, update query where clause to search title and slug.
5. Ensure search is scoped to current user/team.
6. Add test for slug-only match if query tests exist.
7. Manually verify search with title mismatch and slug match.

### Risks and gotchas

- Slug search should not leak hidden/team event types outside permissions.
- Case sensitivity should match title search behavior.
- Debounced searches may be handled by TanStack Query/tRPC cache.

### Suggested PR shape

- Search predicate/query update.
- Minimal UI change if needed.
- Test for slug match.

## Story 32: Add booking form unsaved-changes warning

### User story

As an attendee, I want a warning before leaving a partially completed booking form so that I do not lose entered information.

### Product scope

Show browser-level unsaved changes warning when the attendee has typed into the booking form but has not submitted. Do not persist draft data.

### First mental model

This is Booker client UI behavior. It should not touch booking APIs.

### Files to inspect first

- `apps/web/modules/bookings/components/Booker.tsx`
- `apps/web/modules/bookings/components/BookEventForm/BookEventForm.tsx`
- existing unsaved-change hooks/utilities in `apps/web` or `packages/lib`

### Implementation plan

1. Search `rg "beforeunload|unsaved|dirty" apps/web packages`.
2. Use existing unsaved changes hook if present.
3. In `BookEventForm.tsx`, rely on React Hook Form dirty state.
4. Enable warning only when form is dirty and not submitting/submitted.
5. Disable warning after successful booking, redirect, or payment handoff.
6. Test leaving untouched form, dirty form, and submitted form.

### Risks and gotchas

- Browser beforeunload messages are not fully customizable.
- Embed flows may handle navigation differently.
- Do not block payment redirects or success redirects.

### Suggested PR shape

- Small hook usage/addition.
- Booker form state integration.
- Manual verification in browser.

## Story 33: Add team member timezone column

### User story

As a team admin, I want to see each member's timezone in the team member list so that I can understand coverage across regions.

### Product scope

Display timezone in the team member list. Do not change member invites or permissions.

### First mental model

This is team settings read UI. Data is on `User.timeZone`, but query selection may need updating.

### Files to inspect first

- `apps/web/modules/teams`
- `packages/trpc/server/routers/viewer/teams` or related team routers
- `packages/features/membership`
- `packages/prisma/schema.prisma`

### Implementation plan

1. Search `rg "team members|Membership|members" apps/web packages/trpc packages/features/membership`.
2. Find team member list query.
3. Confirm whether member user `timeZone` is selected.
4. Add `timeZone` to safe selection if missing.
5. Add a column/cell using existing table patterns.
6. Add fallback for missing timezone.
7. Verify regular member and admin views.

### Risks and gotchas

- Team member data is permission-scoped.
- Responsive table layouts can get crowded.
- Use display formatting if a timezone helper exists.

### Suggested PR shape

- Query selection update.
- Team member table UI update.
- Permission-aware manual check.

## Story 34: Add "last synced" text to selected calendars

### User story

As an organizer, I want to see when a selected calendar last synced so that I can detect stale calendar data.

### Product scope

Display existing sync timestamp if available. Do not implement sync scheduling.

### First mental model

This is selected calendar metadata display. Check `SelectedCalendar` fields before assuming the data exists.

### Files to inspect first

- `packages/prisma/schema.prisma`
- `packages/features/selectedCalendar`
- `packages/trpc/server/routers/viewer/calendars`
- calendar settings UI in `apps/web`

### Implementation plan

1. Inspect `SelectedCalendar` model fields.
2. Search `rg "lastSync|synced|channel" packages/features apps/web packages/trpc`.
3. If a timestamp exists, include it in the query.
4. If no timestamp exists, write a separate migration/design story rather than faking it.
5. Render relative time using existing date helpers.
6. Add fallback like "Not synced yet".
7. Manually verify connected calendar rows.

### Risks and gotchas

- Calendar sync fields may mean channel sync, cache sync, or provider watch sync. Use the right one.
- Do not expose credential details.
- Relative time should respect locale where practical.

### Suggested PR shape

- Data selection update.
- UI display.
- No sync behavior change.

## Story 35: Add safer error message for failed booking payment

### User story

As an attendee, I want a clear and safe error message when payment setup fails so that I know whether to retry or contact the organizer.

### Product scope

Improve the user-facing error copy for payment failures during booking creation. Do not change payment provider behavior.

### First mental model

This is booking submission error handling, likely in `useBookings.ts` and payment branches of `RegularBookingService`.

### Files to inspect first

- `apps/web/modules/bookings/hooks/useBookings.ts`
- `packages/features/bookings/lib/service/RegularBookingService.ts`
- payment integration code under `packages/app-store`
- translation files under `packages/i18n`

### Implementation plan

1. Search `rg "payment" apps/web/modules/bookings packages/features/bookings packages/app-store`.
2. Identify errors thrown by payment setup.
3. Map technical payment errors to safe user-facing messages in client mutation error handling or server error codes.
4. Add/update translation keys.
5. Preserve detailed errors in logs/Sentry, not attendee UI.
6. Test payment failure path with mocked provider failure if existing tests support it.

### Risks and gotchas

- Payment errors may contain sensitive provider details.
- Do not swallow errors needed for observability.
- Paid bookings may have redirect flows separate from normal success.

### Suggested PR shape

- Error mapping/copy update.
- i18n keys.
- Test or manual mocked failure verification.

## Story 36: Add a "copy attendee email" action on booking details

### User story

As an organizer, I want to copy an attendee's email from booking details so that I can quickly contact them.

### Product scope

Add a copy-to-clipboard action beside attendee email on booking details. Do not add messaging.

### First mental model

This is booking detail UI only if attendee email is already loaded.

### Files to inspect first

- `apps/web/modules/bookings`
- booking detail components
- clipboard helper patterns in `apps/web`
- `packages/ui`

### Implementation plan

1. Search `rg "attendee.email|Attendee|navigator.clipboard|copy" apps/web/modules/bookings apps/web packages/ui`.
2. Find where attendee emails render.
3. Add an icon button with tooltip using existing UI components.
4. Copy the email to clipboard.
5. Show success/failure toast.
6. Do not expose the action where attendee email is hidden or unavailable.
7. Test manually in booking details.

### Risks and gotchas

- Clipboard API can fail on insecure origins.
- Mobile layout should not crowd attendee rows.
- Avoid adding duplicate copy controls for multiple attendees without clear labels.

### Suggested PR shape

- One UI component update.
- Existing clipboard/toast helper use.
- Manual verification.

## Story 37: Add "event type has no destination calendar" warning

### User story

As an organizer, I want to be warned when an event type has no destination calendar so that bookings are not missed on my calendar.

### Product scope

Show a non-blocking warning in event type settings when no event-type or user-level destination calendar is configured.

### First mental model

This is event type settings UI plus calendar configuration data.

### Files to inspect first

- `apps/web/modules/event-types`
- `packages/features/calendars`
- `packages/trpc/server/routers/viewer/calendars`
- `packages/prisma/schema.prisma`

### Implementation plan

1. Search `rg "DestinationCalendar|destinationCalendar" apps/web packages/features packages/trpc`.
2. Determine fallback behavior: event type destination vs user default destination.
3. Find event type settings data query.
4. Include enough data to know whether a destination exists.
5. Render a warning banner with link to calendar settings.
6. Keep it non-blocking unless product asks otherwise.
7. Test with user default present, event type override present, and neither present.

### Risks and gotchas

- Some integrations may create references without a destination calendar.
- Team event types may use host calendars.
- Avoid implying bookings will fail if the system still has fallback behavior.

### Suggested PR shape

- Query/data update.
- Settings warning UI.
- Manual scenarios.

## Story 38: Add route-level loading skeleton for event type settings

### User story

As an organizer, I want event type settings to show a structured loading skeleton so that the page feels stable while data loads.

### Product scope

Add loading skeleton UI only. Do not change data fetching.

### First mental model

This is frontend UX. Find existing skeleton components and route-level loading patterns.

### Files to inspect first

- `apps/web/modules/event-types`
- `apps/web/app`
- `packages/ui`
- existing `Skeleton.tsx` files

### Implementation plan

1. Search `rg "Skeleton" apps/web/modules packages/ui apps/web/app`.
2. Find the event type settings route/component.
3. Identify current loading state.
4. Build skeleton using existing skeleton primitives.
5. Match layout dimensions to avoid content jump.
6. Verify desktop and mobile.
7. No backend tests needed; consider visual/manual check.

### Risks and gotchas

- Skeleton should not hide real error states.
- Avoid duplicating a generic skeleton component if one exists.
- Keep dimensions stable.

### Suggested PR shape

- One loading component.
- Route/component loading state use.
- Manual screenshot verification.

## Story 39: Add admin-only user lookup by email

### User story

As an admin, I want to look up a user by email so that I can troubleshoot account-specific issues faster.

### Product scope

Add admin-only search by exact email. Return minimal user metadata. Do not build broad fuzzy user search.

### First mental model

This is admin tRPC with strict permission checks and careful data selection.

### Files to inspect first

- `packages/trpc/server/routers/viewer/admin`
- `packages/features/users`
- `packages/prisma/schema.prisma`
- admin UI modules under `apps/web`

### Implementation plan

1. Search `rg "admin.*users|watchlist|role" packages/trpc apps/web packages/features`.
2. Find admin permission middleware/pattern.
3. Add exact-email input schema with Zod email validation.
4. Query user by email with minimal safe selection.
5. Add UI form in admin area.
6. Show not-found state without leaking extra detail to non-admins.
7. Test non-admin rejection, invalid email, not found, found.

### Risks and gotchas

- User lookup is sensitive; permission checks are mandatory.
- Avoid returning password, tokens, credentials, or private metadata.
- Exact search is safer than broad search for first version.

### Suggested PR shape

- Admin tRPC procedure.
- Minimal admin UI.
- Permission tests.

## Story 40: Add local dev checklist command documentation

### User story

As a new developer, I want a documented checklist of commands to run before opening a PR so that I can catch common issues early.

### Product scope

Documentation only. Add a PR checklist section to onboarding docs with commands for type-check, lint, tests, and focused E2E guidance.

### First mental model

This is docs-only. Use what we learned from `package.json`, `turbo.json`, `vitest.workspace.ts`, and `playwright.config.ts`.

### Files to inspect first

- `package.json`
- `turbo.json`
- `vitest.workspace.ts`
- `playwright.config.ts`
- `docs/onboarding/07-local-dev.md`
- `docs/onboarding/06-patterns.md`

### Implementation plan

1. Confirm current root scripts.
2. Add a "Before opening a PR" section to `07-local-dev.md` or a new onboarding checklist doc.
3. Include common commands:
   - `yarn type-check`
   - `yarn lint`
   - `yarn test`
   - focused package tests where appropriate
   - `yarn test-e2e` for browser-flow changes
4. Explain when to run each command.
5. Include docs-only exception language.
6. No app tests required for the docs PR itself.

### Risks and gotchas

- Commands can drift; keep docs tied to root scripts.
- Do not imply every PR must run full E2E locally if the repo does not require it.
- Keep guidance practical for junior developers.

### Suggested PR shape

- One docs update.
- No code changes.
- Manual review for command accuracy.

## Story 41: Show API key last-used time in developer settings

### User story

As a developer, I want to see when each API key was last used so that I can rotate or delete stale credentials.

### Product scope

Display the existing `ApiKey.lastUsedAt` value in the API keys list. Do not change API key authentication, hashing, or usage tracking in this story.

### First mental model

This is a developer settings read UI. The database already has `lastUsedAt`, and the API key list query appears to select it.

### Files to inspect first

- `apps/web/modules/settings/developer/api-keys-view.tsx`
- `apps/web/modules/api-keys/api-keys/components/ApiKeyListItem.tsx`
- `packages/trpc/server/routers/viewer/apiKeys/list.handler.ts`
- `packages/features/api-keys-legacy/api-keys/repositories/PrismaApiKeyRepository.ts`
- `packages/prisma/schema.prisma`

### Implementation plan

1. Confirm `ApiKey.lastUsedAt` exists in Prisma and is selected by `PrismaApiKeyRepository.findApiKeysFromUserId`.
2. Add a small row of metadata to `ApiKeyListItem.tsx` beside the existing expiration text.
3. Render relative time with the same dayjs/i18n pattern already used for expiration.
4. Show a clear fallback such as "Never used" when `lastUsedAt` is null.
5. Avoid showing the hashed key, raw key, or any additional secret material.
6. Add a component test if API key list item tests exist; otherwise manually verify keys with null and non-null `lastUsedAt`.

### Risks and gotchas

- `lastUsedAt` may not update for all auth paths yet. Do not imply perfect audit coverage unless usage tracking is verified.
- Dates should be readable across locales and time zones.
- Keep the list compact; API key settings should remain scannable.

### Suggested PR shape

- API key list item UI update.
- Translation/fallback copy.
- Manual verification with used and unused keys.

## Story 42: Require a reason when adding blocklist entries

### User story

As a trust and safety admin, I want every manually added blocklist entry to include a reason so that future reviewers understand why the entry exists.

### Product scope

Require a non-empty reason when adding a user or email to the organization or system blocklist. Do not change booking report review flows beyond the same validation.

### First mental model

This is a blocklist form validation and mutation-contract story. The UI already has a reason field, but the server should enforce the rule too.

### Files to inspect first

- `apps/web/modules/blocklist/components/CreateBlocklistEntryModal.tsx`
- `apps/web/modules/blocklist/components/BookingReportDetailsModal.tsx`
- `packages/features/blocklist/types.ts`
- `packages/trpc/server/routers/viewer/admin/watchlist`
- `packages/features/watchlist/lib/service/AdminWatchlistQueryService.ts`
- `packages/features/watchlist/lib/repository/WatchlistRepository.ts`

### Implementation plan

1. Find the mutation used by `CreateBlocklistEntryModal` and `BookingReportDetailsModal`.
2. Add client-side validation to require `reason.trim().length > 0`.
3. Add server-side Zod validation for the create/add-to-blocklist mutation.
4. Normalize the reason by trimming it at the boundary.
5. Preserve existing permission checks for organization vs system scope.
6. Add tests for missing reason, whitespace reason, and valid reason.

### Risks and gotchas

- System blocklist entries are higher impact than organization entries. Make sure both paths enforce the rule.
- Booking report actions may reuse the same add-to-blocklist flow but pass slightly different payloads.
- Do not expose internal reviewer notes to blocked bookers.

### Suggested PR shape

- Form validation update.
- Server schema/handler validation update.
- Focused tests for both blocklist scopes.

## Story 43: Add search to the holidays settings list

### User story

As an organizer, I want to search holidays by name so that I can quickly enable or disable the specific holidays that affect my availability.

### Product scope

Add client-side filtering to the holidays settings list. Do not change holiday syncing, caching, country selection, or booking conflict behavior.

### First mental model

This is a settings UI enhancement. The holidays query already returns the list for the selected country, so the first version can filter in React.

### Files to inspect first

- `apps/web/modules/settings/my-account/holidays-view.tsx`
- `packages/trpc/server/routers/viewer/holidays/_router.ts`
- `packages/trpc/server/routers/viewer/holidays/getUserSettings.handler.ts`
- `packages/lib/holidays`
- `packages/features/holidays/repositories/HolidayRepository.ts`

### Implementation plan

1. Find where `settings.holidays` is rendered in `HolidaysView`.
2. Add local search state near the country selector/list controls.
3. Filter by holiday name with case-insensitive matching.
4. Keep the enabled/disabled state and toggle mutation behavior unchanged.
5. Show an empty state when the search has no matches.
6. Verify that conflict warnings still use the full settings/conflict query, not the filtered visual list.

### Risks and gotchas

- Search should not accidentally hide conflict warnings for disabled/enabled holidays.
- Country changes should reset or preserve search intentionally; choose the behavior that matches nearby settings search patterns.
- The list may be localized later, so filter on the displayed name, not only an internal ID.

### Suggested PR shape

- One UI state/filter update.
- Empty-state copy.
- Manual verification for matched, unmatched, and country-change states.

## Story 44: Show active travel schedule context in availability settings

### User story

As a frequent traveler, I want my availability settings to show which travel schedule affects a date override so that I can understand why times appear in a different timezone.

### Product scope

Display a small contextual label for date overrides that fall inside a travel schedule. Do not change slot calculation or travel schedule creation.

### First mental model

Travel schedules already affect schedule date ranges. This story is read-only UI that connects the schedule editor to the existing travel schedule data.

### Files to inspect first

- `apps/web/modules/availability/[schedule]/schedule-view.tsx`
- `packages/features/schedules/components/DateOverrideList.tsx`
- `packages/features/schedules/lib/date-ranges.ts`
- `packages/features/travelSchedule/repositories/TravelScheduleRepository.ts`
- `packages/trpc/server/routers/viewer/travelSchedules/getTravelSchedules.handler.ts`
- `packages/prisma/schema.prisma`

### Implementation plan

1. Confirm `schedule-view.tsx` passes travel schedules into schedule components only for the default schedule.
2. Inspect how `DateOverrideList.tsx` currently receives and uses `travelSchedules`.
3. Add a label for overrides whose date range intersects a travel schedule.
4. Include the travel timezone and date range, not internal IDs.
5. Keep the label read-only and avoid mutating override values.
6. Add a focused component test if `DateOverrideList` has nearby tests; otherwise manually verify overrides inside and outside travel schedule ranges.

### Risks and gotchas

- Travel schedules may have no end date.
- Overlaps should be computed with dayjs in the same timezone style as existing schedule code.
- Event-type-specific schedules may not use travel schedules the same way as the default schedule.

### Suggested PR shape

- Date override display update.
- Small helper for overlap detection if needed.
- Manual verification across active, future, and open-ended travel schedules.

## Story 45: Add preferred segment persistence to the bookings table

### User story

As an organizer, I want my bookings table to reopen on my preferred saved segment so that I do not have to reapply the same filters every day.

### Product scope

Use the existing filter segment preference infrastructure for the bookings table. Do not create new filter types in this story.

### First mental model

This is data-table state plus tRPC segment preferences. The table may already support saved segments; the work is to verify bookings table wiring and persist the preferred segment.

### Files to inspect first

- `apps/web/modules/bookings/views/bookings-view.tsx`
- `apps/web/modules/data-table/DataTableProvider.tsx`
- `apps/web/modules/data-table/contexts/DataTableSegmentContext.tsx`
- `apps/web/modules/data-table/hooks/useSegments.ts`
- `packages/trpc/server/routers/viewer/filterSegments/_router.tsx`
- `packages/features/data-table/repositories/filterSegment.ts`
- `packages/prisma/schema.prisma`

### Implementation plan

1. Identify the bookings table `tableIdentifier`.
2. Confirm the bookings table is wrapped with `DataTableProvider` and segment support.
3. Check whether `preferredSegmentId` from `filterSegments.list` is consumed.
4. If missing, wire bookings table initialization to the preferred segment for that table.
5. Call `setPreference` when the user intentionally selects or clears a saved segment.
6. Preserve URL `segment` query param behavior; explicit URLs should win over stored preferences.
7. Add tests around preference precedence: URL segment, preferred segment, no preference.

### Risks and gotchas

- Segment preferences are table-specific; do not leak a team members segment into bookings.
- Team segments may have permissions and admin-only visibility.
- Avoid saving preferences on transient invalid segment IDs.

### Suggested PR shape

- Bookings table segment wiring.
- Preference precedence test or component-level test.
- Manual verification with personal and team segments.

## Story 46: Harden OAuth client redirect URI validation

### User story

As a platform admin, I want OAuth client redirect URIs to be restricted to safe URL schemes so that unsafe redirects cannot be approved or used during authorization.

### Product scope

Validate OAuth redirect URIs as `http` or `https`, with production rules favoring `https`. Do not change OAuth consent, token exchange, or client secret behavior.

### First mental model

This is a security boundary. Redirect URI validation exists in tRPC schemas and API v2 platform inputs, but `z.string().url()` alone may allow schemes product does not want.

### Files to inspect first

- `apps/web/modules/settings/oauth/view/OAuthClientFormFields.tsx`
- `packages/trpc/server/routers/viewer/oAuth/submitClientForReview.schema.ts`
- `packages/trpc/server/routers/viewer/oAuth/updateClient.schema.ts`
- `packages/trpc/server/routers/viewer/oAuth/getClientForAuthorization.schema.ts`
- `packages/features/oauth/services/OAuthService.ts`
- `apps/api/v2/src/modules/oauth-clients`
- `packages/prisma/schema.prisma`

### Implementation plan

1. Find every schema that accepts or validates an OAuth redirect URI.
2. Add a shared safe URL helper if there is no existing one.
3. Allow only `https:` by default, and allow `http://localhost` or configured local dev origins only if local/dev behavior needs it.
4. Reuse the same validation for submit, update, authorization, and API v2 client creation/update.
5. Add clear form errors in the developer OAuth client UI.
6. Add tests for `https`, localhost dev, `javascript:`, `data:`, malformed URLs, and mismatched redirect URIs.

### Risks and gotchas

- Existing approved clients may already have `http` redirects. Decide whether to grandfather existing values or require reapproval.
- Authorization flow must compare the requested URI against the stored URI after normalization rules are clear.
- Avoid breaking local Playwright OAuth tests without intentionally updating their fixtures.

### Suggested PR shape

- Shared redirect URL validation helper.
- tRPC/API v2 schema updates.
- OAuth flow tests and UI form error copy.

## Story 47: Add a test web push notification action

### User story

As a user, I want to send myself a test browser notification so that I can confirm notifications are working before relying on them for booking updates.

### Product scope

Add a test action for the current user's web push subscription. Do not create new notification types or change booking notification behavior.

### First mental model

This is an authenticated tRPC mutation that reuses existing web push subscription and delivery code.

### Files to inspect first

- `apps/web/modules/notifications/components/WebPushContext.tsx`
- `packages/features/notifications/sendNotification.ts`
- `packages/trpc/server/routers/viewer/_router.tsx`
- notification subscription procedures currently exposed through `loggedInViewerRouter`
- `packages/prisma/schema.prisma`

### Implementation plan

1. Locate the existing add/remove notification subscription mutations.
2. Add an authenticated `sendTestNotification` mutation near the subscription procedures.
3. Load only the current user's active subscription.
4. Reuse `sendNotification` with a safe test title/body and no sensitive booking data.
5. Add a UI button where notification permission/subscription state is managed.
6. Handle browser permission denied, missing subscription, and VAPID-not-configured cases gracefully.
7. Test mutation behavior with a mocked subscription and mocked `sendNotification`.

### Risks and gotchas

- Web push can fail due to browser, OS, or service worker state even when server code is correct.
- Do not expose another user's push subscription.
- Missing VAPID keys should produce a useful settings error, not a crash.

### Suggested PR shape

- tRPC mutation.
- Small notification settings UI action.
- Tests with mocked send behavior.

## Story 48: Show Cal Video recording count on booking details

### User story

As an organizer, I want booking details to show whether a Cal Video meeting has recordings so that I can find follow-up material quickly.

### Product scope

Display recording count for Cal Video bookings when the booking has a Cal Video room name. Do not add playback or transcript viewing in this story.

### First mental model

This is booking detail UI backed by the existing Cal Video recording tRPC procedure.

### Files to inspect first

- `apps/web/modules/bookings/components/BookingDetailsSheet.tsx`
- `apps/web/modules/bookings/views/bookings-single-view.tsx`
- `packages/trpc/server/routers/viewer/calVideo/_router.tsx`
- `packages/trpc/server/routers/viewer/calVideo/getCalVideoRecordings.handler.ts`
- `packages/trpc/server/routers/viewer/calVideo/getCalVideoRecordings.schema.ts`
- `packages/features/conferencing/lib/videoClient`

### Implementation plan

1. Find where booking metadata or booking references expose the Cal Video room/meeting identifier.
2. Add a conditional query for `trpc.viewer.calVideo.getCalVideoRecordings`.
3. Render a small count in booking details only when the booking uses Cal Video.
4. Show loading and error states without blocking the rest of booking details.
5. Avoid querying recordings for non-Cal Video locations.
6. Add a component test for zero, one, multiple, and unavailable recording states if practical.

### Risks and gotchas

- Recording APIs may be slow or unavailable. Keep the query optional and non-blocking.
- Permission checks should ensure only authorized organizers can see recording metadata.
- Room names and meeting URLs can differ; use the same identifier expected by `getCalVideoRecordings`.

### Suggested PR shape

- Booking details conditional query.
- Small recording count display.
- Manual verification with Cal Video and non-video bookings.

## Story 49: Add busy-source legend to the troubleshooter calendar

### User story

As a support engineer, I want the troubleshooter calendar to label why time is busy so that I can explain unavailable slots without inspecting raw calendar data.

### Product scope

Add a legend or labels for busy source categories already returned to the troubleshooter. Do not change slot calculation or expose private event details.

### First mental model

The troubleshooter already fetches busy times and renders them on a large calendar. This story is presentation plus possibly a small data-shape enrichment if source type is missing.

### Files to inspect first

- `apps/web/modules/troubleshooter/components/LargeCalendar.tsx`
- `apps/web/modules/troubleshooter/components/CalendarToggleContainer.tsx`
- `packages/features/troubleshooter/store.ts`
- `packages/trpc/server/routers/viewer/slots/util.ts`
- `packages/trpc/server/routers/viewer/availability`
- `packages/features/busyTimes/services/getBusyTimes`

### Implementation plan

1. Inspect the busy event shape consumed by `LargeCalendar.tsx`.
2. Identify whether each busy item already carries source information, calendar ID, or a generic title.
3. If source exists, map it to labels such as calendar event, booking limit, out of office, reserved slot, or date override.
4. If source does not exist, add the smallest safe metadata field at the tRPC boundary used by the troubleshooter only.
5. Render a compact legend and keep individual busy events private by default.
6. Test with selected calendars, existing bookings, OOO, and reserved slots if fixtures exist.

### Risks and gotchas

- Busy calendar events can contain sensitive titles. Keep labels generic unless data is already intentionally exposed.
- Troubleshooter mode is controlled by `_enableTroubleshooter`; do not add extra payload to normal slot calls.
- Color and legend categories should remain stable across reloads.

### Suggested PR shape

- Troubleshooter data-shape audit.
- Legend UI.
- Tests or manual fixtures for multiple busy sources.

## Story 50: Show custom destination-calendar reminder in event type settings

### User story

As an organizer, I want event type settings to show the custom calendar reminder for the selected destination calendar so that I know what reminder guests and hosts will see on calendar events.

### Product scope

Display and allow editing of the existing `DestinationCalendar.customCalendarReminder` value from event type settings. Do not change calendar event creation behavior.

### First mental model

Destination calendar data is already loaded for event type settings, and reminder persistence has a calendar tRPC handler. The work is to connect the existing setting to the event type settings UI.

### Files to inspect first

- `apps/web/modules/event-types/components/tabs/advanced/EventAdvancedTab.tsx`
- `packages/features/calendars/components/DestinationCalendarSelector.tsx`
- `packages/trpc/server/routers/viewer/calendars/setDestinationReminder.handler.ts`
- `packages/features/calendars/repositories/DestinationCalendarRepository.ts`
- `packages/trpc/server/routers/viewer/calendars/connectedCalendars.handler.ts`
- `packages/prisma/schema.prisma`

### Implementation plan

1. Confirm whether `connectedCalendars.handler.ts` returns `destinationCalendar.customCalendarReminder`.
2. Find any existing UI for destination reminder settings outside event type settings.
3. Add a small reminder control near `DestinationCalendarSelector` in `EventAdvancedTab`.
4. Use the existing `setDestinationReminder` mutation rather than adding a second persistence path.
5. Support clearing the reminder back to null.
6. Invalidate connected calendar/event type settings queries after save.
7. Test user-level and event-type-level destination calendars separately.

### Risks and gotchas

- Destination calendars can be user-level or event-type-level. Make sure the mutation updates the intended row.
- Reminder units must be clear: minutes before the event, not an absolute timestamp.
- Do not imply external providers support every reminder value if the calendar integration clamps values later.

### Suggested PR shape

- Query data confirmation/update.
- Event type settings control.
- Mutation wiring and focused tests.

## How to use these stories for practice

For each story, ask the junior developer to produce a one-page design note before coding:

1. What files will you touch?
2. Which existing flow does this extend?
3. Is the API tRPC, Next API, or API v2?
4. Which Prisma models are involved?
5. What validation is needed?
6. What auth/permission checks are needed?
7. What tests prove the change?
8. What is the smallest safe PR?

Then compare their plan with the relevant docs:

- `docs/onboarding/03-architecture.md`
- `docs/onboarding/04-key-flows.md`
- `docs/onboarding/05-data-models.md`
- `docs/onboarding/06-patterns.md`

The goal is not to memorize paths. The goal is to learn how to move from a user-facing request to the right layer of the codebase with confidence.
