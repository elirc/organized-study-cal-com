# Hard Feature Implementation Journal

This journal records a second learning spike: I temporarily built 11 of the more difficult practice stories, studied the affected boundaries, and then reverted the source code so the repository behavior stays unchanged. The value here is the trail of decisions: where I entered the codebase, which files formed the path, and why each change belongs at that layer.

The spike touched API v2 controllers, tRPC routers, Prisma-backed handlers, React settings components, shared webhook DTOs, and app-store UI. That spread is the point: harder Cal.com work usually crosses one or more package boundaries, so a good implementation plan starts by naming the ownership boundary before editing.

## Stories Built In The Spike

- Story 6: Add helper text to custom booking fields
- Story 11: Show invalid calendar credential warnings in settings
- Story 13: Add API v2 endpoint to list current user's event types
- Story 15: Add a "copy schedule" action
- Story 18: Add attendee locale to booking webhook payloads
- Story 21: Add no-show reason when marking attendee no-show
- Story 24: Add integration category badges in app-store settings
- Story 27: Add selected-calendar count to event type settings
- Story 29: Add "booking source" filter to bookings list
- Story 34: Add "last synced" text to selected calendars
- Story 39: Add admin-only user lookup by email

## Story 6: Helper Text For Custom Booking Fields

### Implementation Path

- `packages/prisma/zod-utils.ts`
- `apps/web/modules/event-types/components/tabs/advanced/FormBuilder.tsx`
- `apps/web/modules/form-builder/components/FormBuilderField.tsx`

### What I Changed

I added `helperText: z.string().optional()` to the shared booking-field Zod shape in `packages/prisma/zod-utils.ts`. That is the important architectural move: event type booking fields are JSON-ish configuration values, so the safest way to let a new property travel through the app is to teach the shared parser about it.

Then I added `InputField` controls in `FormBuilder.tsx` in both normal custom fields and field variants. The form builder already uses `field` and `variant` edit dialogs, so the feature belongs beside `placeholder`, not in a separate state store.

Finally, I rendered `field.helperText` in `FormBuilderField.tsx` below the field control with `text-subtle mt-1 text-sm`. This keeps the display behavior close to where booking field UI is already interpreted.

### Architecture Lesson

This is a typical "configuration data" change. The path is:

```text
Event type settings form
  -> bookingFields JSON validated by zod-utils
  -> stored on EventType
  -> public booking form field renderer
```

The non-obvious bit is that a UI-only field still needs schema support because booking fields are shared across modules and may be parsed outside the immediate settings component.

### Tests I Would Add Before Shipping

- A focused unit test for the booking field schema accepting `helperText`.
- A component test around `FormBuilderField` rendering helper text only when present.
- A regression test that existing fields without `helperText` still render normally.

## Story 11: Invalid Calendar Credential Warnings

### Implementation Path

- `packages/features/users/types/invalidAppCredentials.ts`
- `packages/trpc/server/routers/viewer/me/checkForInvalidAppCredentials.ts`
- `apps/web/modules/users/components/InvalidAppCredentialsBanner.tsx`

### What I Changed

The handler already returned app-level invalid credential metadata. I expanded the result shape with `invalidCredentialCount?: number`, selected credential IDs from Prisma, grouped invalid credentials by `appId`, and returned a count per app.

In `InvalidAppCredentialsBanner.tsx`, I threaded that count into the banner text so multiple broken credentials for the same app can be distinguished without changing the UI flow.

### Architecture Lesson

This is a small example of "backend summary, frontend presentation." The frontend should not re-fetch credentials or infer hidden details. The tRPC handler owns the user-specific query and returns the exact display-ready summary the banner needs.

```text
settings page
  -> trpc.viewer.me.checkForInvalidAppCredentials
  -> Credential rows grouped by appId
  -> InvalidAppCredentialsBanner
```

### Tests I Would Add Before Shipping

- Handler test with two invalid credentials for one app and one for another app.
- UI test for singular and plural display states.

## Story 13: API V2 Current User Event Type Summaries

### Implementation Path

- `apps/api/v2/src/platform/event-types/event-types_2024_06_14/controllers/event-types.controller.ts`
- `apps/api/v2/src/platform/event-types/event-types_2024_06_14/services/event-types.service.ts`

### What I Changed

I added a guarded `GET /v2/event-types/me` endpoint before the dynamic `GET /:eventTypeId` route. That ordering matters. In Nest, a dynamic route like `/:eventTypeId` can swallow literal paths if the literal route is declared later.

The controller used existing API v2 patterns:

- `@Permissions([EVENT_TYPE_READ])`
- `@UseGuards(ApiAuthGuard)`
- `@ApiHeader(API_KEY_OR_ACCESS_TOKEN_HEADER)`
- `SUCCESS_STATUS`

The service method `getCurrentUserEventTypeSummaries(userId)` calls the existing repository method and returns a minimal shape: `id`, `title`, `slug`, `length`, and `hidden`.

### Architecture Lesson

API v2 work should stay in the versioned platform module. Do not route this through web tRPC just because tRPC already has event type procedures. External API behavior is owned by `apps/api/v2`.

```text
external API client
  -> Nest controller
  -> ApiAuthGuard + PermissionsGuard
  -> EventTypesService_2024_06_14
  -> EventTypesRepository_2024_06_14
```

### Tests I Would Add Before Shipping

- API e2e test that `/me` returns only the authenticated user's event types.
- Regression test that `/123` still resolves to `getEventTypeById`.
- OpenAPI/schema update if this endpoint needs generated platform types.

## Story 15: Custom Name When Duplicating A Schedule

### Implementation Path

- `packages/trpc/server/routers/viewer/availability/schedule/duplicate.schema.ts`
- `packages/trpc/server/routers/viewer/availability/schedule/duplicate.handler.ts`

### What I Changed

I extended `ZScheduleDuplicateSchema` with optional `name: z.string().trim().min(1).optional()` and changed `duplicate.handler.ts` to use `input.name ?? `${schedule.name} (Copy)``.

I deliberately kept the fallback behavior. Existing callers that only pass `scheduleId` should keep working.

### Architecture Lesson

Schedule duplication is a tRPC mutation with a clear validation boundary. The handler should trust parsed input, but the schema decides what shape input may take.

```text
client mutation
  -> duplicate.schema.ts
  -> duplicate.handler.ts
  -> prisma.schedule.create
```

### Tests I Would Add Before Shipping

- Schema test rejecting empty names.
- Handler test proving the fallback name still works.
- Handler test proving a custom name is persisted.

## Story 18: Attendee Locale In Booking Webhooks

### Implementation Path

- `packages/features/webhooks/lib/factory/versioned/v2021-10-20/BookingPayloadBuilder.ts`

### What I Changed

The webhook payload builder already enriches attendees with `utcOffset`, `firstName`, and `lastName`. I added `locale: a.language?.locale` inside the same attendee mapping.

### Architecture Lesson

Webhook payload shape is versioned. The right place for payload transformation is not the booking service or calendar event producer; it is the versioned payload builder. That keeps each webhook version stable and reviewable.

```text
booking event DTO
  -> BookingPayloadBuilder for v2021-10-20
  -> attendees enriched for outbound webhook
```

### Tests I Would Add Before Shipping

- Payload builder test with attendee `language.locale`.
- Payload builder test where attendee language is absent.
- Contract review because adding fields to webhooks can affect consumers.

## Story 21: No-Show Reason In Webhook Payloads

### Implementation Path

- `packages/features/webhooks/lib/types/params.ts`
- `packages/features/webhooks/lib/dto/types.ts`
- `packages/features/webhooks/lib/sendPayload.ts`
- `packages/features/webhooks/lib/service/BookingWebhookService.ts`
- `packages/features/webhooks/lib/factory/versioned/v2021-10-20/BookingPayloadBuilder.ts`

### What I Changed

I widened the no-show attendee shape from `{ email; noShow }` to `{ email; noShow; reason?: string | null }` across params, DTOs, and payload types. In `BookingWebhookService.emitBookingNoShow`, I normalized missing reasons to `null`. In the versioned payload builder, I mapped each attendee again to guarantee the emitted payload has a predictable `reason` key.

### Architecture Lesson

This was the most "type propagation" oriented story. A field added to webhook input must move through several layers:

```text
caller
  -> BookingNoShowParams
  -> BookingWebhookService
  -> BookingNoShowDTO
  -> versioned BookingPayloadBuilder
  -> outbound webhook payload
```

The risk is partial propagation: adding the type in one file but losing the field before the final payload.

### Tests I Would Add Before Shipping

- Service test that `emitBookingNoShow` preserves `reason`.
- Builder test that missing `reason` becomes `null`.
- Snapshot/contract test for the webhook payload.

## Story 24: Integration Category Badges

### Implementation Path

- `packages/features/apps/components/AppList.tsx`

### What I Changed

I imported `Badge` from `@calcom/ui/components/badge` and rendered category badges inside `ChildAppCard` before `AppSettings`. The category list used `[item.category, ...(item.categories ?? [])].filter(Boolean)` so both the primary legacy category and multi-category metadata can appear.

### Architecture Lesson

The app-store settings page is already passed app metadata from `getConnectedApps`. This is a presentational change, so it should not fetch app metadata again or create new backend fields unless the existing `AppCardApp` data is insufficient.

### Tests I Would Add Before Shipping

- Component test that categories render for a connected app.
- Visual check for apps with many categories.
- Deduping logic if `category` also appears inside `categories`.

## Story 27: Selected-Calendar Count

### Implementation Path

- `packages/trpc/server/routers/viewer/calendars/connectedCalendars.handler.ts`

### What I Changed

I enriched the connected calendar handler return type with:

- `selectedCalendarCount: number`
- `lastSyncAt: string | null`

The handler queried `prisma.selectedCalendar.groupBy({ by: ["integration"], where: { userId, eventTypeId }, _count, _max: { updatedAt } })`, built a `Map` by integration, and attached the count to each connected calendar.

### Architecture Lesson

Connected calendar display data is assembled server-side because it merges integration metadata with user-specific selected-calendar rows. That makes the tRPC handler the right place to compute counts.

```text
calendar settings UI
  -> trpc.viewer.calendars.connectedCalendars
  -> getConnectedDestinationCalendarsAndEnsureDefaultsInDb
  -> SelectedCalendar grouped by integration
  -> enriched connectedCalendars response
```

### Tests I Would Add Before Shipping

- Handler test for user-level selected calendars where `eventTypeId` is `null`.
- Handler test for event-type-level selected calendars.
- UI test only after the client renders the count.

## Story 29: Booking Source Filter

### Implementation Path

- `packages/trpc/server/routers/viewer/bookings/get.schema.ts`
- `packages/trpc/server/routers/viewer/bookings/get.handler.ts`

### What I Changed

I imported `CreationSource` from `@calcom/prisma/enums`, added `creationSource?: CreationSource` to the filter type, and added `creationSource: z.nativeEnum(CreationSource).optional()` to the schema.

In `get.handler.ts`, I added a Kysely `where("Booking.creationSource", "=", filters.creationSource)` filter inside the shared query-filter mapping.

### Architecture Lesson

Bookings list filtering is not a single query; it builds several access-scoped booking queries and applies shared filters to each query before unioning them. New filters usually belong in the `queriesWithFilters` mapping so every access path receives the same narrowing behavior.

### Tests I Would Add Before Shipping

- Schema test accepting valid `CreationSource` enum values.
- Handler/query test verifying the filter is applied to all unioned booking query paths.
- UI integration test if a dropdown is added.

## Story 34: Last Synced Text For Selected Calendars

### Implementation Path

- `packages/trpc/server/routers/viewer/calendars/connectedCalendars.handler.ts`

### What I Changed

This was implemented with Story 27 because the same `SelectedCalendar.groupBy` query can return `_max.updatedAt`. I exposed that as `lastSyncAt` on each connected calendar.

### Architecture Lesson

When two UI stories need the same backing data, prefer one backend enrichment instead of two separate queries. In a real PR I would document the combined backend change and keep the UI commits separate if that made review easier.

### Tests I Would Add Before Shipping

- Handler test that `lastSyncAt` is ISO formatted.
- Handler test where no selected calendars exist and the value is `null`.
- Timezone/display test in the consuming component once UI is added.

## Story 39: Admin-Only User Lookup By Email

### Implementation Path

- `packages/trpc/server/routers/viewer/admin/lookupUserByEmail.schema.ts`
- `packages/trpc/server/routers/viewer/admin/lookupUserByEmail.handler.ts`
- `packages/trpc/server/routers/viewer/admin/_router.ts`

### What I Changed

I added a small schema with `email: z.string().email()`, a handler that lowercases the email and calls `ctx.prisma.user.findUnique`, and a new `lookupUserByEmail` query on `adminRouter` using `authedAdminProcedure`.

The selected user fields were intentionally limited:

- `id`
- `username`
- `name`
- `email`
- `timeZone`
- `role`
- `locked`
- `verified`
- `createdDate`

### Architecture Lesson

Admin procedures should live under the admin router so permission checks are inherited from `authedAdminProcedure`. The handler should not return the full `User` row just because Prisma can. Admin tools need least-privilege response shapes too.

```text
admin UI/client
  -> trpc.viewer.admin.lookupUserByEmail
  -> authedAdminProcedure
  -> lookupUserByEmail.handler.ts
  -> prisma.user.findUnique
```

### Tests I Would Add Before Shipping

- Procedure test proving non-admin callers are rejected by the admin procedure.
- Handler test for case-insensitive lookup input.
- Handler test that sensitive fields are not returned.

## Cross-Feature Notes

The harder stories fell into four recurring implementation patterns:

1. Extend a typed contract first.

   Examples: `ZScheduleDuplicateSchema`, booking filter schema, webhook DTO types. In Cal.com, many runtime bugs are prevented by letting Zod and TypeScript define the crossing point between layers.

2. Keep versioned API behavior versioned.

   API v2 changes belong in `apps/api/v2/src/platform/...`, not in web tRPC. Webhook payload changes belong in `packages/features/webhooks/lib/factory/versioned/...`, not in a generic service.

3. Add data where the owner already has context.

   The connected calendar handler already knows the authenticated user, event type ID, and Prisma client, so selected-calendar counts belong there. The invalid credential handler already owns credential lookup, so it should compute counts.

4. Preserve old callers.

   Optional input fields such as schedule duplicate `name` should keep previous behavior when omitted.

## Revert Checklist

After documenting the spike, I reverted all source changes and removed temporary files. The source files affected by the spike were:

- `packages/prisma/zod-utils.ts`
- `apps/web/modules/event-types/components/tabs/advanced/FormBuilder.tsx`
- `apps/web/modules/form-builder/components/FormBuilderField.tsx`
- `packages/features/users/types/invalidAppCredentials.ts`
- `packages/trpc/server/routers/viewer/me/checkForInvalidAppCredentials.ts`
- `apps/web/modules/users/components/InvalidAppCredentialsBanner.tsx`
- `apps/api/v2/src/platform/event-types/event-types_2024_06_14/controllers/event-types.controller.ts`
- `apps/api/v2/src/platform/event-types/event-types_2024_06_14/services/event-types.service.ts`
- `packages/trpc/server/routers/viewer/availability/schedule/duplicate.schema.ts`
- `packages/trpc/server/routers/viewer/availability/schedule/duplicate.handler.ts`
- `packages/features/webhooks/lib/factory/versioned/v2021-10-20/BookingPayloadBuilder.ts`
- `packages/features/webhooks/lib/dto/types.ts`
- `packages/features/webhooks/lib/service/BookingWebhookService.ts`
- `packages/features/webhooks/lib/types/params.ts`
- `packages/features/webhooks/lib/sendPayload.ts`
- `packages/features/apps/components/AppList.tsx`
- `packages/trpc/server/routers/viewer/calendars/connectedCalendars.handler.ts`
- `packages/trpc/server/routers/viewer/bookings/get.schema.ts`
- `packages/trpc/server/routers/viewer/bookings/get.handler.ts`
- `packages/trpc/server/routers/viewer/admin/_router.ts`
- `packages/trpc/server/routers/viewer/admin/lookupUserByEmail.schema.ts`
- `packages/trpc/server/routers/viewer/admin/lookupUserByEmail.handler.ts`

## Remaining Stories For The Junior Developer

Sorted from more approachable to more complex:

1. Story 40: Add local dev checklist command documentation
2. Story 38: Add route-level loading skeleton for event type settings
3. Story 23: Add default phone country selection on booking forms
4. Story 26: Add "hide weekends" preference to the booker
5. Story 32: Add booking form unsaved-changes warning
6. Story 8: Show the organizer timezone on public booking pages
7. Story 20: Add booking title max-length validation
8. Story 35: Add safer error message for failed booking payment
9. Story 37: Add "event type has no destination calendar" warning
10. Story 14: Add filtering to the admin watchlist page
11. Story 33: Add team member timezone column
12. Story 9: Require cancellation reason for organizer cancellations
13. Story 2: Add a confirmation toggle to event type creation
14. Story 1: Add an internal note field to booking creation
15. Story 25: Add email template preview for booking confirmation
16. Story 5: Add a "copy private booking link" action for hashed links
17. Story 22: Add org slug to public profile metadata
18. Story 19: Add visible retry status for webhook deliveries
19. Story 10: Add a "resend confirmation email" action for bookings
20. Story 4: Add a new webhook event for booking internal-note updates
21. Story 28: Add audit entry when event type location changes
22. Story 3: Show "out of office" blocked reason in slot debugging logs
