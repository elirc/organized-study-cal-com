# Language & Runtime Model (JS/Node)

## The event loop, from first principles

Node runs your JavaScript on a single thread. Anything CPU-bound blocks that thread and therefore every concurrent request on the process. I/O (DB queries, `fetch`, file reads) is handed to the OS/libuv and its callback is queued — so while a booking awaits a DB write, the loop is free to progress other requests. Promise callbacks (microtasks) drain before the next timer/IO callback (macrotask).

Where this repo uses it well: heavy or failure-prone work is pushed **off** the request path onto the tasker (`packages/features/tasker/task-processor.ts:11-40`), so a slow email send never blocks a booking response. The booking handler is almost entirely `await`ed I/O (`RegularBookingService.ts:523-629`) — no CPU-bound loops on the hot path.

Sharp edges here: the slots computation (`slots/util.ts`) does real work per request; that's why it's cached in Redis for 2 seconds (`:79`, `:104-120`) — not to unblock the loop, but to avoid recomputing. Don't confuse "cache for latency" with "offload for the loop."

Pitfall checklist:
- [ ] No synchronous heavy loops on a request path (parse/crypto/large maps).
- [ ] `await` in a `for` loop only when serialization is required (else `Promise.all`).
- [ ] Long/failable side effects go to the tasker, not inline.

Drill: read `task-processor.ts:20-37` and explain why running task handlers concurrently (`allSettled`) keeps throughput up without extra threads.

Interview angle: [08-interview-prep/01](../08-interview-prep/01-js-ts-node-deep-dive.md) Q1.

## Promises, async/await, serial vs parallel

`async`/`await` is sugar over promises. Sequential `await`s add latency; independent awaits should run together. Failure semantics differ: `Promise.all` rejects on first failure (`reserveSlot.handler.ts:79`, intentional atomicity); `Promise.allSettled` never rejects and reports per-item status (`task-processor.ts:37`, intentional batch isolation).

Sharp edge: JS has no cancellation — `Promise.all` rejecting doesn't stop the other in-flight promises. The other upserts in the reserve handler still complete.

Drill: find one serial `await` pair in `RegularBookingService.ts:523-549` that must be serial (data dependency) and argue whether any pair could be parallelized.

Interview angle: [08-interview-prep/01](../08-interview-prep/01-js-ts-node-deep-dive.md) Q2, Q11.

## Closures

A closure captures its lexical scope at creation. Used deliberately in `withSlotsCache` (`slots/util.ts:104-120`), which closes over `redisClient` and the wrapped function. The failure version is React's stale closure (an effect/callback capturing an old state/prop) — same mechanism, different outcome.

Pitfall checklist:
- [ ] In HOFs, know what you're capturing (usually fine, long-lived).
- [ ] In components, know that each render creates new closures over that render's values.

Drill: contrast the safe module-singleton closure in `di/containers/AvailableSlots.ts:22` with a hypothetical stale-closure bug in a Booker callback.

Interview angle: [08-interview-prep/01](../08-interview-prep/01-js-ts-node-deep-dive.md) Q3.

## Modules, imports, and the runtime graph

ESM modules with a strict convention here: `import type` for types (erased at build, avoids runtime cycles and side-effect imports) and **no barrel imports** (`AGENTS.md`: import from `@calcom/ui/components/button`, not `@calcom/ui`). Barrels + value imports create fat dependency graphs and slow builds.

Sharp edge: with `isolatedModules`, mixing type and value imports can break single-file transpilation — separating them is required, not cosmetic. Also: importing a module for a type alone (as a value import) pulls its side effects into the bundle.

Drill: open `RegularBookingService.ts:18-116` and count `import type` vs value imports; explain what would change at runtime if line 30 (`import type { BookingEmailAndSmsTasker }`) dropped `type`.

Interview angle: [08-interview-prep/01](../08-interview-prep/01-js-ts-node-deep-dive.md) Q7.

## Node vs. browser vs. edge

Server code may use `node:*` (e.g. `createHmac` in `sendPayload.ts:1`), secrets, and the DB; client code (`"use client"`) may not. The webhook signature *must* be server-side. The App Router page runs on the server and serializes props to the client Booker — Dates become strings across that hop (`create-booking.ts:8-12`).

Drill: explain why `createWebhookSignature` (`sendPayload.ts:296`) cannot move into the Booker.

Interview angle: [08-interview-prep/01](../08-interview-prep/01-js-ts-node-deep-dive.md) Q13.

## Errors at runtime

Domain code throws typed errors (`HttpError`/`ErrorWithCode`), tRPC throws `TRPCError`, and a boundary translates unknown → transport (`getServerErrorFromUnknown`, `errorConversionMiddleware.ts`). `withReporting` (`sentryWrapper.ts:236`) captures with context and **rethrows** — never swallows. Contrast the misleading swallow at `reserveSlot.handler.ts:104-109`.

Drill: trace how a `HttpError` thrown at `RegularBookingService.ts:589` reaches the HTTP client as a status code (find `defaultResponder`).

Interview angle: [08-interview-prep/01](../08-interview-prep/01-js-ts-node-deep-dive.md) Q9, Q10.
