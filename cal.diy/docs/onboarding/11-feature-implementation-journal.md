# Feature Implementation Journal

This journal documents a learning spike where seven practice user stories from `docs/onboarding/10-practice-user-stories.md` were actually built out, inspected, and then removed from the codebase so the product behavior remains unchanged.

The goal was not to ship these features. The goal was to practice moving through the Cal.diy architecture like a developer would on a real PR:

```text
user story
  -> product scope
  -> route/component/API choice
  -> schema or query boundary
  -> feature package or repository
  -> validation and testing plan
  -> code spike
  -> revert spike
```

## Final state

The code changes from this spike were undone after documentation. Only this journal remains.

The seven stories implemented during the spike were:

1. Story 30: Add lightweight app health check for web
2. Story 12: Add duration preset buttons to event type creation
3. Story 31: Add event type search by slug in dashboard
4. Story 7: Add default before/after buffers for newly created event types
5. Story 16: Add max recurring booking occurrences validation
6. Story 17: Validate success redirect URLs for event types
7. Story 36: Add a "copy attendee email" action on booking details

## Story 30: Add Lightweight App Health Check For Web

### Planning thought process

This story is an app-level HTTP concern, not domain logic. It does not belong in `packages/features`, `packages/trpc`, or API v2. The web app already has App Router API routes under:

- `apps/web/app/api`

So the smallest implementation is a Next.js App Router route handler.

### Files affected in the spike

- `apps/web/app/api/health/route.ts`

### Implementation sketch

The spike added:

```ts
import { NextResponse } from "next/server";

export const dynamic = "force-dynamic";

export function GET() {
  return NextResponse.json({
    status: "ok",
    app: "cal.diy-web",
    timestamp: new Date().toISOString(),
  });
}
```

### Architecture connection

This is intentionally outside tRPC. Health checks are usually called by infrastructure, not by the authenticated React app. Requiring tRPC context, session loading, or Prisma would make the endpoint heavier and less useful as a liveness probe.

### What I would test before shipping

- `GET /api/health` returns `200`.
- Response body includes `status: "ok"`.
- Endpoint does not require authentication.
- Endpoint does not touch the database.

### Why it was removed

The user asked for a learning build-out only. The route was deleted after the spike so it does not add a real public endpoint.

## Story 12: Add Duration Preset Buttons To Event Type Creation

### Planning thought process

Event type creation is a logged-in app flow. The user-facing create form lives in the feature package because it is shared beyond one local web route:

- `packages/features/eventtypes/components/CreateEventTypeForm.tsx`

The existing form already uses React Hook Form, and the `length` field is already validated using:

- `MIN_EVENT_DURATION_MINUTES`
- `MAX_EVENT_DURATION_MINUTES`

That means presets should write into the existing form field rather than introduce new component state.

### Files affected in the spike

- `packages/features/eventtypes/components/CreateEventTypeForm.tsx`

### Implementation sketch

The spike added:

```ts
const DURATION_PRESETS = [15, 30, 45, 60] as const;
const selectedLength = form.watch("length");
```

Then it rendered preset buttons below the existing duration field:

```tsx
{DURATION_PRESETS.map((preset) => (
  <button
    key={preset}
    type="button"
    className={classNames(...)}
    onClick={() => {
      form.setValue("length", preset, {
        shouldDirty: true,
        shouldTouch: true,
        shouldValidate: true,
      });
    }}>
    {preset} {t("minutes").toLowerCase()}
  </button>
))}
```

### Architecture connection

This is UI-only because the backend already accepts `length`. The key architecture lesson is to respect the existing form owner. `CreateEventTypeForm` owns the user input; `useCreateEventType.ts` submits it; the tRPC handler persists it.

The flow remains:

```text
CreateEventTypeForm
  -> useCreateEventType
  -> trpc.viewer.eventTypesHeavy.create
  -> create.handler.ts
  -> EventTypeRepository
  -> EventType.length
```

### What I would test before shipping

- Clicking `15`, `30`, `45`, or `60` updates the form field.
- The custom duration input still works.
- Validation still enforces min/max.
- The selected preset visually follows the current value.

### Why it was removed

The UI change was reverted so event type creation remains unchanged.

## Story 31: Add Event Type Search By Slug In Dashboard

### Planning thought process

The dashboard event type list uses tRPC:

- UI: `apps/web/modules/event-types/views/event-types-listing-view.tsx`
- Query: `trpc.viewer.eventTypes.getEventTypesFromGroup`
- Handler: `packages/trpc/server/routers/viewer/eventTypes/getEventTypesFromGroup.handler.ts`

The existing server query searched only `title`. Since the story asks for dashboard search behavior, the right layer is the tRPC handler query, not client-only filtering. Server-side filtering keeps pagination/infinite query behavior consistent.

### Files affected in the spike

- `packages/trpc/server/routers/viewer/eventTypes/getEventTypesFromGroup.handler.ts`

### Implementation sketch

The spike introduced a shared Prisma condition:

```ts
const searchQueryConditions = searchQuery
  ? {
      OR: [
        { title: { contains: searchQuery, mode: "insensitive" as Prisma.QueryMode } },
        { slug: { contains: searchQuery, mode: "insensitive" as Prisma.QueryMode } },
      ],
    }
  : {};
```

Then it reused that condition for both personal and team event type queries.

### Architecture connection

This handler sits at an important boundary:

```text
React search input
  -> tRPC infinite query
  -> getEventTypesFromGroup.handler.ts
  -> EventTypeRepository
  -> Prisma where clause
```

Changing the handler rather than the UI means the behavior applies consistently to:

- personal event types
- child event types
- team event types
- paginated/infinite query results

### What I would test before shipping

- Searching by title still works.
- Searching by slug works when title does not match.
- Team event types are still permission-scoped.
- Private team behavior still strips users/hosts/children as before.

### Why it was removed

The query change was reverted after the spike.

## Story 7: Add Default Before/After Buffers For New Event Types

### Planning thought process

This is a creation-default story. Since event type creation can be called through tRPC and possibly by multiple frontend surfaces, the default belongs server-side in the create handler, not only in the form.

The relevant server file is:

- `packages/trpc/server/routers/viewer/eventTypes/heavy/create.handler.ts`

### Files affected in the spike

- `packages/trpc/server/routers/viewer/eventTypes/heavy/create.handler.ts`

### Implementation sketch

The spike added constants:

```ts
const DEFAULT_EVENT_TYPE_BEFORE_BUFFER_MINUTES = 5;
const DEFAULT_EVENT_TYPE_AFTER_BUFFER_MINUTES = 5;
```

Then it applied them while building `Prisma.EventTypeCreateInput`:

```ts
const data: Prisma.EventTypeCreateInput = {
  ...rest,
  beforeEventBuffer: rest.beforeEventBuffer ?? DEFAULT_EVENT_TYPE_BEFORE_BUFFER_MINUTES,
  afterEventBuffer: rest.afterEventBuffer ?? DEFAULT_EVENT_TYPE_AFTER_BUFFER_MINUTES,
  ...
};
```

### Architecture connection

The create handler is the bridge between validated tRPC input and Prisma data:

```text
CreateEventTypeForm
  -> useCreateEventType
  -> eventTypesHeavy.create
  -> create.handler.ts
  -> EventTypeRepository.create
  -> EventType.beforeEventBuffer / EventType.afterEventBuffer
```

Placing defaults here protects all callers of the create procedure. If the default lived only in React, API-like callers could still create event types without buffers.

### What I would test before shipping

- Creating an event type without buffer input saves both defaults.
- Creating one with explicit buffer values preserves the explicit values.
- Team event type creation still works.
- Slot availability reflects the buffers after creation.

### Why it was removed

The defaulting behavior was reverted because this was a learning-only implementation.

## Story 16: Add Max Recurring Booking Occurrences Validation

### Planning thought process

Recurring booking creation is a booking creation concern. Booking creation validates request bodies in:

- `packages/features/bookings/lib/bookingCreateBodySchema.ts`

This is a good first boundary because it protects server input before deeper booking orchestration.

### Files affected in the spike

- `packages/features/bookings/lib/bookingCreateBodySchema.ts`

### Implementation sketch

The spike added:

```ts
export const MAX_RECURRING_BOOKING_OCCURRENCES = 52;
```

Then it constrained:

```ts
recurringCount: z.number().max(MAX_RECURRING_BOOKING_OCCURRENCES).optional()
```

and:

```ts
allRecurringDates: z.array(...).max(MAX_RECURRING_BOOKING_OCCURRENCES).optional()
```

### Architecture connection

The booking architecture has several layers:

```text
Booker form
  -> booking-to-mutation-input-mapper
  -> /api/book/event
  -> bookingCreateBodySchema
  -> RegularBookingService
  -> createBooking persistence
```

The schema is an early guard. In a shipping PR, I would also add a service-level guard in `RegularBookingService` because booking service code can be reached by more than one transport path.

### What I would test before shipping

- `52` recurring dates passes.
- `53` recurring dates fails.
- `recurringCount` over limit fails.
- Regular non-recurring bookings are unaffected.
- Error message is clear enough for the booking UI.

### Why it was removed

The schema constraint was reverted so recurring booking behavior remains unchanged.

## Story 17: Validate Success Redirect URLs For Event Types

### Planning thought process

Redirect URLs are security-sensitive. The relevant field appears in both event type creation and event type update paths:

- create schema: `packages/features/eventtypes/lib/schemas.ts`
- update schema: `packages/trpc/server/routers/viewer/eventTypes/types.ts`

Because create and update are separate schemas, both need validation or the system becomes inconsistent.

### Files affected in the spike

- `packages/features/eventtypes/lib/schemas.ts`
- `packages/trpc/server/routers/viewer/eventTypes/types.ts`

### Implementation sketch

The spike added a Zod helper in both schema files:

```ts
const safeSuccessRedirectUrl = z
  .string()
  .trim()
  .url()
  .refine((url) => ["http:", "https:"].includes(new URL(url).protocol), {
    message: "Success redirect URL must use http or https",
  });
```

Then it changed:

```ts
successRedirectUrl: z.string().nullable().optional()
```

to:

```ts
successRedirectUrl: safeSuccessRedirectUrl.nullable().optional()
```

For create, the spike also added `successRedirectUrl?: string | null` to `TCreateEventTypeInput`.

### Architecture connection

Event type settings flow is schema-heavy:

```text
settings/create UI
  -> event type schema
  -> tRPC handler
  -> EventTypeRepository / Prisma
  -> bookingSuccessRedirect uses saved value after booking
```

The important idea is that redirect safety should be enforced before persistence. The booking success page later consumes this value, so allowing unsafe strings into the database creates downstream risk.

### What I would test before shipping

- `https://example.com/thanks` passes.
- `http://example.com/thanks` passes if product allows HTTP.
- `javascript:alert(1)` fails.
- malformed URLs fail.
- empty/null values still behave as before.

### Why it was removed

The validation changes were reverted after documenting the implementation.

## Story 36: Add Copy Attendee Email Action On Booking Details

### Planning thought process

This is a UI-only booking details enhancement. The data already exists in `booking.attendees`. No API change is needed if attendee emails are already loaded.

The relevant component is:

- `apps/web/modules/bookings/components/BookingDetailsSheet.tsx`

The repo already has a copy helper:

- `@calcom/lib/hooks/useCopy`

So the spike reused that helper instead of calling `navigator.clipboard` directly.

### Files affected in the spike

- `apps/web/modules/bookings/components/BookingDetailsSheet.tsx`

### Implementation sketch

The spike imported:

```ts
import { useCopy } from "@calcom/lib/hooks/useCopy";
```

Then inside `WhoSection`:

```ts
const { copyToClipboard } = useCopy();
```

And beside each attendee email:

```tsx
<Button
  color="minimal"
  size="icon"
  StartIcon="clipboard"
  aria-label={t("copy_to_clipboard")}
  onClick={() => copyToClipboard(attendee.email)}
/>
```

### Architecture connection

This lives entirely in the React presentation layer:

```text
BookingDetailsSheet
  -> booking.attendees from existing booking data
  -> useCopy helper
  -> clipboard
```

Since no data is mutated and no new server data is needed, adding tRPC would be unnecessary. The right implementation is close to where the email is rendered.

### What I would test before shipping

- Button appears beside attendee email.
- Clicking copies the right email.
- Multiple attendees copy independently.
- UI remains readable on narrow screens.
- Clipboard failure uses existing `useCopy` failure behavior.

### Why it was removed

The UI action was reverted after the spike.

## Cross-feature lessons

### Transport matters

The seven stories used different transport/layer choices:

- health check: public App Router API route
- event type creation UI: shared feature component
- event type search/defaults/redirect validation: tRPC + schema/handler layer
- recurring booking validation: booking API schema layer
- copy attendee email: client-only UI

Choosing the right layer is the main skill.

### Schema boundaries are leverage points

Two stories were mostly schema work:

- recurring booking max occurrences
- success redirect URL validation

In this repo, Zod schemas are where a lot of API safety is centralized. A junior developer should learn to ask, "Where is this input validated?" before changing a handler.

### Feature packages hold product meaning

Event type behavior lives in `packages/features/eventtypes`; booking behavior lives in `packages/features/bookings`. The web app imports and composes those features, but should not become a dumping ground for domain rules.

### Keep UI-only changes UI-only

The duration presets and copy-email action did not need backend changes. Adding backend code where it is not needed makes reviews harder and increases test burden.

### Reverting the spike is part of the lesson

After building the features, the code changes were removed. That creates a safe teaching artifact:

- the repo behavior stays unchanged
- the journal preserves the reasoning
- future developers can compare the plan to the implementation sketch without inheriting half-approved product changes

## Revert checklist used

After writing this journal, I removed the spike code by reverting/deleting only the files touched by the spike:

- deleted `apps/web/app/api/health/route.ts`
- reverted `apps/web/modules/bookings/components/BookingDetailsSheet.tsx`
- reverted `packages/features/bookings/lib/bookingCreateBodySchema.ts`
- reverted `packages/features/eventtypes/components/CreateEventTypeForm.tsx`
- reverted `packages/features/eventtypes/lib/schemas.ts`
- reverted `packages/trpc/server/routers/viewer/eventTypes/getEventTypesFromGroup.handler.ts`
- reverted `packages/trpc/server/routers/viewer/eventTypes/heavy/create.handler.ts`
- reverted `packages/trpc/server/routers/viewer/eventTypes/types.ts`

The only intended remaining artifact is this journal.
