# Remaining Stories Implementation Journal

This journal closes the practice-story set. I temporarily spiked the remaining stories, then restored source code afterward so the repository stays unchanged for learning. The point is not that every spike line is production-ready. The point is to show how a senior engineer thinks through placement, data ownership, syntax choices, and review risk in this codebase.

The remaining stories were:

- Story 1: Add an internal note field to booking creation
- Story 2: Add a confirmation toggle to event type creation
- Story 3: Show "out of office" blocked reason in slot debugging logs
- Story 4: Add a new webhook event for booking internal-note updates
- Story 5: Add a "copy private booking link" action for hashed links
- Story 8: Show the organizer timezone on public booking pages
- Story 9: Require cancellation reason for organizer cancellations
- Story 10: Add a "resend confirmation email" action for bookings
- Story 14: Add filtering to the admin watchlist page
- Story 19: Add visible retry status for webhook deliveries
- Story 20: Add booking title max-length validation
- Story 22: Add org slug to public profile metadata
- Story 23: Add default phone country selection on booking forms
- Story 25: Add email template preview for booking confirmation
- Story 26: Add "hide weekends" preference to the booker
- Story 28: Add audit entry when event type location changes
- Story 32: Add booking form unsaved-changes warning
- Story 33: Add team member timezone column
- Story 35: Add safer error message for failed booking payment
- Story 37: Add "event type has no destination calendar" warning
- Story 38: Add route-level loading skeleton for event type settings
- Story 40: Add local dev checklist command documentation

## Story 1: Internal Note On Booking Creation

### Spike Files

- `packages/features/bookings/lib/bookingCreateBodySchema.ts`
- `packages/features/bookings/lib/handleNewBooking/createBooking.ts`
- `packages/features/bookings/lib/service/RegularBookingService.ts`

### What I Implemented

I added `internalNote: z.string().trim().max(2000).optional()` to `bookingCreateBodySchema`.

The syntax matters:

- `z.string()` says the API only accepts text.
- `.trim()` normalizes user input at the boundary rather than letting trailing whitespace leak into persistence.
- `.max(2000)` gives the field a clear product and storage boundary.
- `.optional()` keeps old booking clients working.

Then I passed `reqBody.internalNote` into `Prisma.BookingCreateInput` in `createBooking.ts`.

### Why This Layer

Booking creation enters through the public booking API path, not the dashboard tRPC path. The schema file is the first trustworthy boundary. `createBooking.ts` is the persistence boundary where request data becomes a `Booking` row.

```text
Booker form
  -> bookingCreateBodySchema
  -> RegularBookingService
  -> buildNewBookingData
  -> Booking.internalNote
```

### Production Follow-Up

The spike only wired the backend. A production version should add a form field in `BookEventForm/BookingFields.tsx` or as a configurable booking field, depending on product intent.

## Story 2: Confirmation Toggle On Event Type Creation

### Spike Files

- `packages/features/eventtypes/lib/schemas.ts`
- `packages/platform/atoms/hooks/event-types/private/useCreateEventTypeForm.ts`
- `packages/features/eventtypes/components/CreateEventTypeForm.tsx`

### What I Implemented

I extended `TCreateEventTypeInput` and `createEventTypeInput` with `requiresConfirmation?: boolean`.

I set a default value in `useCreateEventTypeForm`:

```ts
defaultValues: {
  length: 15,
  requiresConfirmation: false,
}
```

That default is important because React Hook Form works best when controlled fields have an initial value. Without it, the checkbox can move through an `undefined -> boolean` transition.

In `CreateEventTypeForm.tsx`, I added a `CheckboxField` bound to `form.watch("requiresConfirmation")` and `form.setValue("requiresConfirmation", checked, { shouldDirty: true })`.

### Why This Layer

The create handler already spreads `...rest` into `Prisma.EventTypeCreateInput`, so once the schema permits `requiresConfirmation`, the handler can persist it without bespoke mapping. That is a good example of using an existing handler shape instead of adding unnecessary glue code.

### Production Follow-Up

Confirm the translation keys exist:

- `requires_confirmation`
- `requires_confirmation_description`

If not, add them to the locale files.

## Story 3: OOO Blocked Reason In Slot Debugging Logs

### Spike Files

- `packages/trpc/server/routers/viewer/slots/util.ts`
- Verified existing reason data in `packages/features/availability/lib/getUserAvailability.ts`

### What I Implemented

The availability layer already carries OOO metadata:

- `reason`
- `emoji`
- `notes`
- `showNotePublicly`

I added debug logging near `getSlots()` so that when a logged/debuggable slot request includes `input.email`, the returned `datesOutOfOffice` object is logged with the slot-building context.

### Why This Layer

The slot router is where availability becomes bookable slots. Logging here answers the real debugging question: "Why did the slot disappear from the booker?"

Logging inside the OOO repository would tell us an OOO row exists. Logging here tells us it affected slot output.

## Story 4: Webhook Event For Internal-Note Updates

### Spike Files

- `packages/prisma/schema.prisma`
- `apps/web/modules/webhooks/components/WebhookForm.tsx`

### What I Implemented

I added `BOOKING_INTERNAL_NOTE_UPDATED` to the Prisma enum and exposed it in `WebhookForm.tsx`.

The UI option used:

```ts
{ value: "BOOKING_INTERNAL_NOTE_UPDATED" as WebhookTriggerEvents, label: "booking_internal_note_updated" }
```

That cast is a spike-only compromise. In production, the generated Prisma enum would include the new value after migration/generation.

### Why This Is Harder

Webhook event work is not just one enum. A complete production implementation needs:

- Prisma enum migration
- generated Prisma client update
- webhook subscriber validation update
- payload DTO
- payload builder support
- event producer when `internalNote` changes
- tests for old webhook versions

This is why a senior engineer starts by listing the contract surfaces before writing the producer.

## Story 5: Copy Private Booking Link

### Spike File

- `apps/web/modules/bookings/views/bookings-single-view.tsx`

### What I Implemented

I added a small `Button` near the booking change actions:

```ts
await navigator.clipboard.writeText(`${window.location.origin}/booking/${bookingInfo.uid}`);
showToast(t("link_copied"), "success");
```

### Why This Syntax

- `navigator.clipboard.writeText` is the browser-native API for clipboard copy.
- `window.location.origin` keeps the link correct across local, preview, and production deployments.
- `showToast` matches existing feedback conventions in this view.

### Production Follow-Up

The story says "private booking link" and mentions hashed links. A real implementation should use the hashed link value if present, not always `/booking/${uid}`.

## Story 8: Organizer Timezone On Public Booking Pages

### Spike Files

- `packages/features/bookings/components/Header.tsx`
- `apps/web/modules/bookings/components/Booker.tsx`

### What I Implemented

I added an optional `organizerTimeZone?: string | null` prop to `Header` and displayed it in the header action area.

In `Booker.tsx`, I passed:

```ts
organizerTimeZone={event.data?.users?.[0]?.timeZone ?? event.data?.user?.timeZone}
```

### Why This Layer

The header already owns date navigation and time-format controls, so timezone context belongs there visually. The parent `Booker` owns event data, so it should pass a simple string rather than making `Header` know the full event shape.

## Story 9: Require Cancellation Reason

### Spike File

- `apps/web/app/api/cancel/route.ts`

### What I Implemented

I added an early guard after parsing `bookingCancelWithCsrfSchema`:

```ts
if (!bookingData.cancellationReason?.trim()) {
  return NextResponse.json(
    { success: false, message: "Cancellation reason is required" },
    { status: 400 }
  );
}
```

### Why This Layer

This route is the public cancellation API. UI validation is helpful, but the server has to enforce it because clients can bypass UI.

The optional chain is intentional: `bookingData.cancellationReason` may be absent on older clients, and `.trim()` should only run when the value exists.

## Story 10: Resend Confirmation Email

### Spike File

- `apps/web/modules/bookings/views/bookings-single-view.tsx`

### What I Implemented

I added a host-only button that shows the expected UI placement and feedback:

```ts
{isHost && !isCancelled && (
  <Button onClick={() => showToast(t("confirmation_email_resent"), "success")}>
    {t("resend_confirmation_email")}
  </Button>
)}
```

### Production Follow-Up

This needs a backend mutation before shipping. The correct production path is likely:

```text
booking detail page
  -> trpc.viewer.bookings.resendConfirmationEmail
  -> authorization check: host/admin only
  -> BookingEmailSmsHandler
  -> confirmation email template
```

The spike UI demonstrates where the action belongs, but production work must not only toast.

## Story 14: Admin Watchlist Filtering

### Spike Files

- `packages/trpc/server/routers/viewer/admin/watchlist/list.schema.ts`
- `packages/features/watchlist/lib/service/AdminWatchlistQueryService.ts`
- `packages/features/watchlist/lib/repository/IWatchlistRepository.ts`
- `packages/features/watchlist/lib/repository/WatchlistRepository.ts`

### What I Implemented

I added filter fields:

- `action: z.nativeEnum(WatchlistAction).optional()`
- `isGlobal: z.boolean().optional()`

Then I threaded them into repository filtering.

### Why This Layer

The tRPC schema is the input contract, the query service owns use-case semantics, and the repository owns Prisma `where` syntax. Keeping those roles separate makes the PR reviewable:

```text
admin UI filters
  -> list.schema.ts
  -> AdminWatchlistQueryService
  -> WatchlistRepository.findAllEntriesWithLatestAudit
```

## Story 19: Visible Retry Status For Webhook Deliveries

### Spike File

- `packages/features/webhooks/lib/handleWebhookScheduledTriggers.ts`

### What I Implemented

I added logging for scheduled webhook delivery attempts:

```ts
logger.info("Webhook delivery attempt finished", {
  scheduledTriggerId: job.id,
  status: response.status,
  ok: response.ok,
});
```

### Why This Is Only A First Step

The story asks for visible retry status. Visibility means persistence plus UI. Logging is useful during the spike, but production should add durable fields or a delivery table rather than relying on logs.

## Story 20: Booking Title Max Length

### Spike File

- `packages/features/bookings/lib/service/RegularBookingService.ts`

### What I Implemented

I added:

```ts
if (typeof evtName === "string" && evtName.length > 120) {
  throw new Error("Booking title must be 120 characters or fewer");
}
```

### Why This Layer

`evtName` is the final event name candidate after dynamic-event handling. Validating here catches both static event names and dynamic titles before calendar creation, webhook payloads, and email subjects consume it.

## Story 22: Org Slug Public Profile Metadata

### Spike File

- `apps/web/server/lib/[user]/getServerSideProps.ts`

### What I Implemented

I enriched profile and entity props with organization slug information:

- `publicProfileOrgSlug: user.profile.organization.slug`
- `orgSlug: currentOrgDomain ?? org?.slug ?? null`

### Why This Layer

This server-side props function is already assembling public profile metadata. Adding org slug here keeps pages from doing follow-up organization lookups on the client.

## Story 23: Default Phone Country Selection

### Spike File

- `apps/web/modules/bookings/components/BookEventForm/BookingFields.tsx`

### What I Implemented

For `field.type === "phone"`, I set:

```ts
field.defaultCountry = field.defaultCountry ?? "US";
```

### Why The Nullish Assignment Shape

`??` preserves an explicit configured default country. It only supplies `"US"` when the field has no default. That matters because event types may eventually define country-specific defaults.

### Production Follow-Up

The default should probably come from locale, profile, or organization settings rather than being hard-coded.

## Story 25: Email Template Preview

### Spike File

- `packages/emails/lib/preview/booking-confirmation-preview.ts`

### What I Implemented

I added a small helper:

```ts
export function buildBookingConfirmationPreview(input) {
  return {
    subject,
    preheader,
    body,
  };
}
```

### Why This Shape

Preview data should be deterministic and side-effect free. It should not send email, require SMTP config, or mutate bookings. A pure function is easy to render in a modal and easy to test.

## Story 26: Hide Weekends Preference

### Spike File

- `packages/features/bookings/components/Header.tsx`

### What I Implemented

I added local `hideWeekends` state and an effect:

```ts
if (day === 6) setSelectedDate({ date: selectedDate.add(2, "day").format("YYYY-MM-DD") });
if (day === 0) setSelectedDate({ date: selectedDate.add(1, "day").format("YYYY-MM-DD") });
```

### Why This Is A Spike

This demonstrates the interaction but is not the final architecture. A real preference should live in the booker store or user preferences so it survives navigation and can affect calendar rendering, not just date navigation.

## Story 28: Audit Entry When Event Type Location Changes

### Spike File

- `packages/trpc/server/routers/viewer/eventTypes/heavy/update.handler.ts`

### What I Implemented

I added structured `logger.info` calls when:

- `destinationCalendar` is changed
- `locations` differ from saved `eventType.locations`

### Why This Layer

The update handler sees both the old event type and the incoming update payload. That is the exact place where audit records should be created, because it can compare before and after values.

### Production Follow-Up

Logs are not an audit trail. A production PR should write to a durable audit table with actor ID, event type ID, old value, new value, and timestamp.

## Story 32: Unsaved Changes Warning

### Spike File

- `apps/web/modules/bookings/components/BookEventForm/BookEventForm.tsx`

### What I Implemented

I added a `beforeunload` listener that checks `bookingForm.formState.isDirty`.

### Why This Syntax

Browser unload confirmation requires:

```ts
event.preventDefault();
event.returnValue = "";
```

The exact message is browser-controlled, but setting `returnValue` still signals that the user has unsaved changes.

## Story 33: Team Member Timezone Column

### Files Verified

- `apps/web/modules/users/components/UsersTable.tsx`

### Finding

The admin users table already has a timezone column:

```tsx
<ColumnTitle>{t("timezone")}</ColumnTitle>
<Cell>{user.timeZone}</Cell>
```

This is a good onboarding moment: sometimes the correct implementation is "do nothing, document that it already exists." Before changing code, search for the expected UI and data shape.

## Story 35: Safer Payment Error Message

### Spike File

- `packages/features/bookings/lib/handlePayment.ts`

### What I Implemented

I wrapped payment provider creation calls in `try/catch`. The provider error is logged internally, but the thrown error becomes:

```ts
throw new Error("Payment could not be completed. Please try another payment method.");
```

### Why This Layer

Payment provider exceptions can include implementation details. `handlePayment.ts` is the boundary between payment vendors and booking creation, so it is the right place to sanitize user-facing errors.

## Story 37: Missing Destination Calendar Warning

### Spike File

- `apps/web/modules/event-types/components/tabs/advanced/EventAdvancedTab.tsx`

### What I Implemented

I added an `Alert` when `calendarsQuery.data?.destinationCalendar` is missing.

### Why This Layer

`EventAdvancedTab` already renders `DestinationCalendarSettings`, so it has the correct data and context. The warning belongs adjacent to the selector because that is where the user can fix it.

## Story 38: Route-Level Loading Skeleton

### File Verified

- `apps/web/app/(use-page-wrapper)/event-types/[type]/loading.tsx`

### Finding

This route already has:

```tsx
import { EventTypeEditPageSkeleton } from "./skeleton";

export default function Loading() {
  return <EventTypeEditPageSkeleton />;
}
```

This story is already solved. The useful lesson is to check App Router `loading.tsx` files before building another loading state inside page components.

## Story 40: Local Dev Checklist Command

### Spike File

- `package.json`

### What I Implemented

I added:

```json
"dev:checklist": "yarn env-check:common && yarn type-check && yarn lint"
```

### Why This Script

It chains the three checks a junior developer should usually run before asking for review:

1. Environment sanity via `env-check:common`
2. TypeScript safety via `type-check`
3. Code style/static lint via `lint`

The `&&` operator is intentional: stop at the first failing check so the developer fixes the most basic problem first.

## Implementation Quality Notes

Some spikes are production-shaped:

- Story 1 backend mapping
- Story 2 create-event-type schema/form path
- Story 9 server-side cancellation guard
- Story 14 watchlist filters
- Story 20 booking title guard
- Story 35 sanitized payment errors
- Story 37 destination calendar warning
- Story 40 dev checklist script

Some spikes are intentionally incomplete because they need broader product or migration work:

- Story 4 needs generated Prisma enums, webhook DTOs, producers, and payload builders.
- Story 10 needs a backend email-resend mutation.
- Story 19 needs persisted delivery status, not only logs.
- Story 25 needs UI and real template integration.
- Story 26 needs persisted booker preference or store-level state.
- Story 28 needs a real audit table write, not logs.

That distinction is part of senior engineering judgment. A teaching spike can show the entry point, but a production PR should tell reviewers which contracts are complete and which are placeholders.

## Source Files Touched During Spike

- `package.json`
- `packages/prisma/schema.prisma`
- `apps/web/app/api/cancel/route.ts`
- `apps/web/modules/bookings/components/BookEventForm/BookEventForm.tsx`
- `apps/web/modules/bookings/components/BookEventForm/BookingFields.tsx`
- `apps/web/modules/bookings/components/Booker.tsx`
- `apps/web/modules/bookings/views/bookings-single-view.tsx`
- `apps/web/modules/event-types/components/tabs/advanced/EventAdvancedTab.tsx`
- `apps/web/modules/webhooks/components/WebhookForm.tsx`
- `apps/web/server/lib/[user]/getServerSideProps.ts`
- `packages/features/bookings/components/Header.tsx`
- `packages/features/bookings/lib/bookingCreateBodySchema.ts`
- `packages/features/bookings/lib/handleNewBooking/createBooking.ts`
- `packages/features/bookings/lib/handlePayment.ts`
- `packages/features/bookings/lib/service/RegularBookingService.ts`
- `packages/features/eventtypes/components/CreateEventTypeForm.tsx`
- `packages/features/eventtypes/lib/schemas.ts`
- `packages/features/watchlist/lib/repository/IWatchlistRepository.ts`
- `packages/features/watchlist/lib/repository/WatchlistRepository.ts`
- `packages/features/watchlist/lib/service/AdminWatchlistQueryService.ts`
- `packages/features/webhooks/lib/handleWebhookScheduledTriggers.ts`
- `packages/platform/atoms/hooks/event-types/private/useCreateEventTypeForm.ts`
- `packages/trpc/server/routers/viewer/admin/watchlist/list.schema.ts`
- `packages/trpc/server/routers/viewer/eventTypes/heavy/update.handler.ts`
- `packages/trpc/server/routers/viewer/slots/util.ts`
- `packages/emails/lib/preview/booking-confirmation-preview.ts`

## Revert Note

After writing this journal, I restored all source files and removed the temporary email preview helper. The onboarding documentation remains, but the application source is back to its original state.
