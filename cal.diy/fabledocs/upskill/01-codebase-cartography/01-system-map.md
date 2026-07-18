# System Map

Cal.diy is a **Yarn-workspaces + Turborepo monorepo**. Apps are thin; the domain logic lives in `packages/*`. Anchors verified 2026-07-11.

## Shape

```
cal.diy/
├─ apps/
│  ├─ web/        Next.js app — App Router pages + legacy Pages Router + tRPC client + Booker UI
│  ├─ api/v2/     NestJS platform REST API (versioned, for third parties)
│  └─ docs/       docs site
├─ packages/
│  ├─ prisma/     schema (2,851 lines) + client + migrations + extensions
│  ├─ trpc/       authenticated API surface (routers/viewer/*), middleware, context
│  ├─ features/   THE domain: bookings, availability, webhooks, tasker, auth, eventtypes, ...
│  ├─ app-store/  integrations (Google/Outlook calendar, video, payments, CRM, ...)
│  ├─ lib/        shared utilities (rate limit, errors, tracing, constants)
│  ├─ ui/ coss-ui/ design system components
│  ├─ emails/ sms/ i18n/ dayjs/ kysely/ types/ ...  supporting packages
│  └─ platform/   atoms (embeddable React components) + examples
└─ turbo.json, biome.json, vitest.workspace.ts, playwright.config.ts, docker-compose.yml
```

## Ownership map

| Concern | Lives in | Entry point |
| --- | --- | --- |
| Public booking UI | `packages/features/bookings/Booker/` + `apps/web/modules/bookings/` | `apps/web/app/(booking-page-wrapper)/[user]/[type]/page.tsx` |
| Booking write logic | `packages/features/bookings/lib/service/RegularBookingService.ts` | `apps/web/pages/api/book/event.ts` |
| Availability/slots | `packages/trpc/server/routers/viewer/slots/util.ts` (+ `packages/features/availability/`) | `slots/getSchedule.handler.ts` |
| Authenticated API | `packages/trpc/server/routers/viewer/*` | `packages/trpc/server/routers/_app.ts` (root = `{ viewer }`) |
| Third-party REST API | `apps/api/v2/src/modules/*` | NestJS `app.module.ts` |
| Persistence | `packages/prisma/` | `schema.prisma`, `BookingRepository.ts`, etc. |
| Background work | `packages/features/tasker/` | `task-processor.ts` |
| Integrations | `packages/app-store/<app>/` | per-app `CalendarService.ts` etc. |
| Auth | `packages/features/auth/lib/` | `next-auth-options.ts`, `sessionMiddleware.ts` |

## Public interfaces vs. private internals

- **Public (untrusted) entry points:** the booking page (SSR), `pages/api/book/event.ts`, `pages/api/cancel`, public slot/reserve tRPC procedures, the NestJS platform API. These get bot/rate-limit/validation defenses.
- **Authenticated interfaces:** tRPC `viewer` routers behind `isAuthed` middleware (`sessionMiddleware.ts`). Type-safe, session-scoped.
- **Private internals:** services, repositories, the tasker, DI containers. Reached only through the above; never expose them directly.

## The one-router root

The entire tRPC surface hangs off one root: `packages/trpc/server/routers/_app.ts` exports `appRouter = router({ viewer: viewerRouter })`, and `AppRouter` is the type the client infers from. Every authenticated feature is a sub-router under `viewer/` (bookings, eventTypes, slots, availability, webhook, ...).

## Mental model to hold

Two hot paths dominate everything: the **read path** (availability/slots, cached, public) and the **write path** (booking, defended, transactional). Almost every interview question, ticket, and risk in this curriculum attaches to one of those two. Keep the diagram above open while you read [05-key-flows.md](05-key-flows.md).

Drill: without looking, name which package owns (a) the booking transaction, (b) the slot cache, (c) webhook signing, (d) the auth session. Check against the ownership table.
