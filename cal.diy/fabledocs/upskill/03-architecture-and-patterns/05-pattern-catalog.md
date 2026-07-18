# Pattern Catalog

Sixteen patterns this codebase uses, taught for **recognition** — you should be able to spot each in any repo and name it in an interview. Each card: the problem it solves, its shape, real anchors, why it works here, failure modes, when to use/avoid, and the interview question it answers. Anchors verified 2026-07-11.

---

## Pattern 1: Thin handler → fat service

Problem it solves: keeps transport (HTTP/tRPC) separate from business logic, so logic is testable and reusable across REST, tRPC, and the platform API.

General shape: the route/handler does auth+parse+delegate; a service class does the work.

Real example: `apps/web/pages/api/book/event.ts:49-58` resolves a service and calls `createBooking`; the logic is in `packages/features/bookings/lib/service/RegularBookingService.ts:486`.
Second example: `packages/trpc/server/routers/viewer/slots/getSchedule.handler.ts:5-8` — 8-line handler, all logic in `AvailableSlotsService`.

Why this implementation works: the same `RegularBookingService` backs the web API route and (via DI containers) other callers; the route only handles HTTP concerns (Turnstile, rate limit, session).

Failure modes: "fat handler" drift — validation or business rules leaking back into routes; anemic services that are just pass-throughs. Watch for logic in `pages/api/*` that should be in a service.

Use it when: logic is reused or complex. Avoid it when: a route is a trivial CRUD passthrough and a service adds only ceremony.

Interview angle: "How do you structure a Node backend?" — layers + why. 
Drill: find one `pages/api` route that does too much and name what belongs in a service.

---

## Pattern 2: Dependency injection via a container

Problem it solves: wires many collaborators (repos, services) without hard-coding constructors everywhere; enables test doubles.

General shape: modules register factories under tokens; a container resolves a service and its transitive deps.

Real example: `packages/features/di/containers/AvailableSlots.ts:22-44` — `createContainer()` then ~15 `container.load(TOKEN, module)` calls, exposed via `getAvailableSlotsService()`.
Second example: `packages/features/bookings/di/RegularBookingService.container.ts` (used at `apps/web/pages/api/book/event.ts:3,49`).

Why this implementation works: the service declares an `IBookingServiceDependencies` interface (`RegularBookingService.ts:423-433`); the container supplies concrete implementations, so tests can inject fakes.

Failure modes: mis-wiring (note the **duplicate `busyTimesModule` load** at `AvailableSlots.ts:36-37` — harmless here but exactly how a wrong token slips in); container becomes a god-object; hidden global state (`container` is a module singleton).

Use it when: many services share repos and you test in isolation. Avoid it when: a couple of `new X()` calls would be clearer.

Interview angle: "What is dependency injection and why use it?" — testability + inversion of control, with this container as your example.
Drill: list the dependencies `AvailableSlotsService` needs (see `IAvailableSlotsService`, `slots/util.ts:89-102`) and which are repos vs. services.

---

## Pattern 3: Repository / Service split

Problem it solves: isolates data-access (queries) from business rules, so business logic is DB-shape-independent and queries are centralized.

General shape: `*Repository` = Prisma queries only; `*Service` = orchestration + rules. Per `AGENTS.md`: "Never put business logic in repositories — that belongs in Services."

Real example: `packages/features/bookings/repositories/BookingRepository.ts:340` (`class BookingRepository`) vs. `RegularBookingService.ts`.
Second example: `packages/features/eventtypes/repositories/eventTypeRepository.ts` vs. the event-type usecases.

Why this implementation works: services depend on repository *interfaces* (`IBookingServiceDependencies.bookingRepository`), so a service test uses an in-memory repo.

Failure modes: query logic creeping into services (raw `prisma.x` calls inside a service — the service still does some, e.g. `createBooking.ts:129-136` calls `prisma.credential.findFirstOrThrow` directly); business logic hiding in repositories.

Use it when: the same entity is queried many ways. Avoid it when: one-off scripts.

Interview angle: "Where does business logic go?" 
Drill: find a `prisma.` call inside a service that arguably belongs in a repository.

---

## Pattern 4: Schema-driven input validation (Zod)

Problem it solves: validate untrusted input against a single source of truth, and derive TS types from it.

General shape: define a zod schema, `parse` at the boundary, infer the type with `z.infer`.

Real example: booking body is validated by a schema **built from the event type's `bookingFields`** — `RegularBookingService.ts:540-549` calls `bookingDataSchemaGetter({view, bookingFields})` then `getBookingData({reqBody, eventType, schema})`.
Second example: every tRPC procedure has a `*.schema.ts` (e.g. `slots/reserveSlot.schema.ts`, `eventTypes/delete.schema.ts`).

Why this implementation works: the schema is dynamic — a custom event type with custom questions gets a matching validator, so the contract adapts per event type.

Failure modes: parsing too late (after using raw input); `.passthrough()` letting unknown fields through; drift between the Prisma model and the zod schema (this repo annotates the Prisma schema with `@zod...` directives, e.g. `schema.prisma:867` `bookingResponses`, to keep them aligned).

Use it when: any external input. Avoid it when: the value is already a validated internal type.

Interview angle: "How do you validate API input in TS?" — zod + infer, one boundary.
Drill: open `schema.prisma:156-275` and count the `@zod.import` directives; explain what they generate.

---

## Pattern 5: Cross-cutting behavior via a Prisma client extension

Problem it solves: enforce an invariant on every write without touching every call site.

General shape: `Prisma.defineExtension({ query: { model: { create/update... } } })` intercepts operations.

Real example: `packages/prisma/extensions/booking-idempotency-key.ts:23-54` — on `booking.create` with status ACCEPTED, it computes and sets `idempotencyKey`; on cancel/reject update, it clears it.

Why this implementation works: idempotency is guaranteed regardless of which of the many code paths creates a booking — you cannot forget it.

Failure modes: **invisible control** — a reader at the call site sees `idempotencyKey: null` (`RegularBookingService.ts:204`) and may not realize the extension overwrites it; hard to unit-test in isolation; ordering with other extensions matters.

Use it when: a DB invariant must hold globally. Avoid it when: the rule is local to one flow (put it there, visibly).

Interview angle: "How would you enforce an invariant across many write paths?" — middleware/extension vs. explicit call; tradeoff is discoverability.
Drill: explain what the idempotency key's inputs (`start.end.userId`) dedupe and what they intentionally don't (attendee).

---

## Pattern 6: Idempotency key + unique constraint

Problem it solves: make retries and double-submits safe.

General shape: derive a deterministic key from the request, store it under a unique index; a duplicate collides instead of duplicating.

Real example: `Booking.idempotencyKey String? @unique` (`schema.prisma:855`), populated by Pattern 5.
Second example: `Task @@unique([referenceUid, type])` (`schema.prisma:1846`) dedupes scheduled tasks.

Why this implementation works: the guarantee is at the database, the one place all writers converge.

Failure modes: key too coarse (drops legitimate distinct writes) or too fine (never collides); relying on app-level check-then-insert instead of the constraint (that's a race — see Flow 3).

Use it when: any at-least-once path (payments, webhooks, retries). Avoid it when: writes are naturally unique already.

Interview angle: "How do you make an API idempotent?" — deterministic key + unique index, DB-enforced.
Drill: compare Booking's key strategy to Task's; why does Task include `referenceUid`?

---

## Pattern 7: Optimistic soft-hold (reservation) with TTL

Problem it solves: reduce checkout collisions without hard-locking a resource.

General shape: write an expiring "hold" row, check for others' holds, release on timeout.

Real example: `SelectedSlots` (`schema.prisma:1437-1449`) written by `reserveSlot.handler.ts:77-103` with `releaseAt` (`:29`).

Why this implementation works: holds auto-expire (`releaseAt` + cron cleanup), so an abandoned checkout doesn't block the slot forever.

Failure modes: the check-then-write is **not transactional** (confirmed TOCTOU, Flow 3); holds are advisory, so two people can still collide — the true guard is the booking unique key.

Use it when: you want UX signal ("someone's booking this") but not correctness-critical locking. Avoid it when: you need a real mutex — use a transaction/`FOR UPDATE`/unique constraint.

Interview angle: "How do you handle two users trying to book the same slot?" — soft hold for UX, unique constraint for truth.
Drill: identify which mechanism actually prevents the double-booking and which just improves UX.

---

## Pattern 8: Function wrapper for observability (higher-order function)

Problem it solves: attach error reporting/tracing to functions without editing their bodies.

General shape: `export const f = withReporting(_f, "f")` — a HOF that wraps sync or async fns, captures exceptions with context, rethrows.

Real example: `packages/lib/sentryWrapper.ts:236` `withReporting`; used at `createBooking.ts:94`, `getBookingData.ts:96`, `handleWebhookTrigger.ts:45` (note the `_name` private + public export convention).

Why this implementation works: one wrapper, consistent Sentry context (`functionName`, redacted args) across the codebase; handles promise vs. value transparently.

Failure modes: double-wrapping; losing the original stack; forgetting to rethrow (this one rethrows — good). Redaction bugs can leak PII into error context.

Use it when: consistent cross-cutting instrumentation. Avoid it when: it obscures a hot path or the wrapper's overhead matters.

Interview angle: "What's a higher-order function and where have you used one?" — this wrapper, with the sync/async handling nuance.
Drill: read `sentryWrapper.ts:55-95`; how does it distinguish sync from async return values?

---

## Pattern 9: Background task queue (tasker) with retries

Problem it solves: move slow/failable side effects off the request path and make them durable.

General shape: `create(type, payload, opts)` enqueues; a processor pulls batches, runs handlers, retries with backoff up to `maxAttempts`.

Real example: `packages/features/tasker/tasker.ts:35-41` interface; `task-processor.ts:11-40` loop; `Task` model (`schema.prisma:1825-1850`).

Why this implementation works: two backends (`internal` DB-polling, `redis`) behind one interface (`tasker-factory.ts`); typed payloads via `TaskPayloads` (`tasker.ts:6-27`); `referenceUid` uniqueness prevents duplicate scheduling.

Failure modes: at-least-once delivery → handlers must be idempotent; poison messages retried forever without a dead-letter; DB-polling backend doesn't scale like a real broker.

Use it when: emails, webhooks, syncs, reminders. Avoid it when: the work must be synchronous to the user's response.

Interview angle: "How do you run background jobs?" — queue + retry + idempotent handler + dead-letter.
Drill: what states can a `Task` row be in? (see `attempts`, `succeededAt`, `lastFailedAttemptAt`).

---

## Pattern 10: Signed webhook payloads (HMAC)

Problem it solves: let receivers verify a webhook really came from Cal.diy and wasn't tampered with.

General shape: `HMAC_SHA256(secret, body)` sent as a header; receiver recomputes and compares.

Real example: `packages/features/webhooks/lib/sendPayload.ts:296-299` (`createWebhookSignature`) and `:312-320` (`fetch` with `X-Cal-Signature-256`, `redirect: "manual"`).

Why this implementation works: shared secret per webhook; manual redirect avoids leaking the signed body to a redirect target (SSRF hardening).

Failure modes: signature over the wrong bytes (must sign the exact serialized body sent); `"no-secret-provided"` fallback (line 299) means unsigned delivery if no secret — receivers must reject that; timing-unsafe comparison on the receiver side.

Use it when: any outbound webhook. Avoid it when: you control both ends over a trusted channel (still nice to have).

Interview angle: "How do you secure webhooks?" — HMAC signature + idempotent consumer + allow-listed URLs.
Drill: what must the receiver do to verify, and what's the risk of the no-secret fallback?

---

## Pattern 11: App Router → legacy `getServerSideProps` bridge

Problem it solves: migrate a huge Pages-Router app to App Router incrementally without rewriting every data loader.

General shape: an RSC `page.tsx` builds a legacy context and runs the old `getServerSideProps` through an adapter.

Real example: `apps/web/app/(booking-page-wrapper)/[user]/[type]/page.tsx:15-19` — `withAppDirSsr(getServerSideProps)` + `buildLegacyCtx(headers, cookies, params, searchParams)`.

Why this implementation works: hundreds of existing SSR loaders keep working; the new shell adds metadata/streaming.

Failure modes: double data-fetch across `page` and `generateMetadata` (mitigated by Next request memoization); async `params`/`searchParams` (Next 15) must be awaited (`:19`); legacy ctx shape mismatches.

Use it when: incrementally adopting App Router. Avoid it when: greenfield — use RSC data fetching directly.

Interview angle: "How would you migrate Pages Router to App Router?" — adapter/bridge, incremental, this exact pattern.
Drill: find another route under `apps/web/app/` using `withAppDirSsr`.

---

## Pattern 12: URL as the source of truth for UI state (zustand + query params)

Problem it solves: make client state deep-linkable, shareable, and back-button-safe.

General shape: a store holds view state but mirrors key fields to/from URL query params.

Real example: `packages/features/bookings/Booker/store.ts:442-708` (`createBookerStore`) with `getQueryParam`/`updateQueryParam`/`removeQueryParam` from `./utils/query-param`.

Why this implementation works: selecting a date/time updates the URL, so a refresh or shared link restores the exact booking step.

Failure modes: two sources of truth drifting (store vs. URL); over-syncing causing history spam; SSR/CSR hydration mismatch if the URL isn't read consistently.

Use it when: state should survive reload/sharing (filters, wizards, bookings). Avoid it when: ephemeral UI (hover, focus).

Interview angle: "How do you persist UI state across reloads?" — URL params vs. localStorage vs. store; tradeoffs.
Drill: name one Booker field that lives only in the store and one mirrored to the URL.

---

## Pattern 13: tRPC middleware chain for auth (`isAuthed` → `isAdmin`)

Problem it solves: enforce authentication/authorization once, and refine `ctx` types as you go.

General shape: middleware resolves session → sets typed `ctx.user` → downstream procedures are guaranteed a user; piped middleware adds role checks.

Real example: `packages/trpc/server/middlewares/sessionMiddleware.ts:6-40` — `isAuthed` throws `UNAUTHORIZED` if no user, then `isAdminMiddleware = isAuthed.unstable_pipe(...)` adds a role gate.

Why this implementation works: after `isAuthed`, `ctx.user` is non-null *in the type system*, so handlers don't null-check; Sentry user is set as a side benefit.

Failure modes: authorization (can this user touch *this* resource?) is NOT covered by `isAuthed` — that's per-handler (see Pattern 14 and the Flow 7 stub); role checks that read stale session claims.

Use it when: every authenticated surface. Avoid it when: public procedures — use `publicProcedure`.

Interview angle: "Authentication vs. authorization — where does each live?" — authn in middleware, authz per resource.
Drill: distinguish what `isAuthed` guarantees from what it does NOT.

---

## Pattern 14: Resource-scoped authorization procedure (and how it can rot)

Problem it solves: ensure the caller may act on the *specific* record, not just that they're logged in.

General shape: a procedure loads the target row, checks ownership/role/membership, then `next()`.

Real example: `packages/trpc/server/routers/viewer/eventTypes/util.ts:24-95` (`eventOwnerProcedure`) and `:103-199` (`createEventPbacProcedure`).

Why this implementation works (personal events): loads the event, verifies `event.userId === ctx.user.id` or assignee (`util.ts:151-158`, `72-73`) — a proper object-level check that stops IDOR.

Failure modes: **team-event branch is a stub** — `PermissionCheckService.checkPermission` (`util.ts:15-20`) returns `true`, so role enforcement is bypassed for team events (confirmed; Flow 7). This is the canonical "looks enforced, isn't" bug.

Use it when: any mutation on a user/team-owned record. Avoid it when: N/A — always scope authz to the resource.

Interview angle: "What is IDOR and how do you prevent it?" — load-then-check ownership; show the working personal-event check and the broken team check as a contrast.
Drill: write the real team-role check for `util.ts:159-167`.

---

## Pattern 15: `select` over `include` for least-privilege queries

Problem it solves: fetch only needed columns — faster, and avoids leaking sensitive fields.

General shape: prefer `select: {...}` to `include: {...}`; never select secret columns.

Real example: `reserveSlot.handler.ts:31-33` selects only `{users: {select: {id}}, seatsPerTimeSlot}`; `eventTypes/util.ts:124-144` selects a minimal shape. `AGENTS.md`: "Use `select` instead of `include`" and "Never expose `credential.key`."

Why this implementation works: the query returns exactly the fields the code uses; sensitive fields (`Credential.key`) never enter the result set.

Failure modes: `include` pulling whole relations (over-fetch + accidental exposure); selecting a secret; N+1 when relations are fetched in loops instead of one `select` with nested relations.

Use it when: always, in production queries. Avoid it when: quick local debugging (still don't ship it).

Interview angle: "How do you avoid leaking sensitive data / over-fetching?" — explicit `select`, least privilege.
Drill: find an `include` in a hot query and convert it (in your head) to a `select`.

---

## Pattern 16: Guard-chain with early returns (state-before-permission)

Problem it solves: make complex preconditions readable and correctly ordered.

General shape: a sequence of `if (bad) throw` checks, ordered so cheap/state checks precede expensive/permission checks; early returns keep nesting flat.

Real example: `handleCancelBooking.ts:122-176` — already-cancelled → disallowed → reason-required → ended → seated-host. `AGENTS.md` explicitly favors early returns.

Why this implementation works: each guard has one reason to reject and one status code; the order prevents acting on already-terminal state before checking permission.

Failure modes: wrong order (checking permission before state, doing work between guards); duplicated guards across call sites instead of one boundary; guards that mutate.

Use it when: any handler with multiple preconditions. Avoid it when: a single condition — don't ceremonialize.

Interview angle: "How do you keep a handler with many checks readable?" — ordered guard clauses, early return, one boundary.
Drill: reorder the cancel guards wrongly and describe the bug each reorder causes.
