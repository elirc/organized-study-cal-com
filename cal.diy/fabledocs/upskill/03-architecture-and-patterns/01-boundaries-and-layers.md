# Boundaries & Layers

## The layers

```
HTTP / tRPC transport   → apps/web/pages/api/*, packages/trpc/server/routers/*
   parse, authn, rate-limit, translate errors — NO business logic
Service (orchestration) → packages/features/*/lib/service/*, *Service.ts
   business rules, transactions boundaries, side-effect triggering
Repository (data access)→ packages/features/*/repositories/*Repository.ts
   Prisma queries only — NO business logic (AGENTS.md)
Persistence            → packages/prisma (schema, client, extensions)
Integrations           → packages/app-store/* (calendars, video, payments)
Async                  → packages/features/tasker (jobs), webhooks
```

## What each layer owns — and must not own

| Layer | Owns | Must NOT own |
| --- | --- | --- |
| Transport | HTTP/tRPC concerns, auth check, input parse, error→status | business rules, DB queries |
| Service | orchestration, invariants, transaction scope, deciding side effects | raw transport types, HTTP status codes deep in logic |
| Repository | queries, `select` shapes | business decisions, cross-entity rules |
| Persistence | schema, constraints, cross-cutting extensions | app logic beyond invariants |

The rule from `AGENTS.md`: "Never put business logic in repositories — that belongs in Services" and "Use `ErrorWithCode` in services; `TRPCError` only in tRPC routers." These *are* the boundary definitions.

## Good boundaries in this repo

- **Thin handler → fat service:** `apps/web/pages/api/book/event.ts` (transport only) delegates to `RegularBookingService` (all logic). The same service backs multiple callers via DI.
- **Repository isolation:** `BookingRepository.ts:340` holds queries; the service depends on the repository *interface* (`IBookingServiceDependencies`, `RegularBookingService.ts:423-433`), so services are testable with fakes.
- **Error translation boundary:** services throw `HttpError`/`ErrorWithCode`; `defaultResponder`/`errorConversionMiddleware` translate to transport at the edge. One translation point.

## Boundary leaks worth noticing (teaching examples)

- **Service reaching into Prisma directly:** `createBooking.ts:129-136` calls `prisma.credential.findFirstOrThrow` inside what is otherwise service/write logic — arguably a repository's job. A defensible pragmatic choice, but a leak to flag in review.
- **Transport concern in a handler that's really logic:** `reserveSlot.handler.ts` mixes cookie-setting (transport) with reservation logic (service) in one function — because it's a tRPC handler doing both. Fine, but it's why the file is hard to unit-test.
- **Authz stub crossing into the transport layer:** the PBAC check lives in a tRPC `util.ts` (transport-ish) but is meant to encode a domain permission — and it's a stub (`eventTypes/util.ts:15-20`). The leak here is *responsibility*: authorization logic that should be a real, tested service is a placeholder in the router layer.

## The senior noticing

When you read a file, ask: *what layer is this, and is it doing another layer's job?* A booking service that formats HTML, a repository that decides whether a user may act, a handler that runs a transaction — each is a boundary leak with a cost (harder to test, reuse, or reason about). Cal.diy is mostly clean; the leaks above are the interesting exceptions.

Drill: classify each of these into a layer and say if it leaks — `event.ts` rate-limit call; `createBooking.ts` credential lookup; `reserveSlot.handler.ts` cookie set; `checkForConflicts.ts`. Self-grade — **Strong:** you name the layer, the leak (if any), and the cost.

Interview angle: "How do you structure a backend and keep layers honest?" → the table above + one real leak with its cost. See [08-interview-prep/03](../08-interview-prep/03-api-and-data-modeling-questions.md).
