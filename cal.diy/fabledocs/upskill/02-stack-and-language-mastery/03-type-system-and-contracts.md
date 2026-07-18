# Type System & Contracts (TypeScript + Zod + tRPC)

## The core idea: types as enforced contracts

In this repo, types aren't decoration — they encode contracts and invariants that the compiler enforces. The chain runs: Prisma schema → generated types → zod schemas (validation) → tRPC inference → client types. One source of truth per layer; casts break the chain (`as any` is banned, `AGENTS.md`).

## Inference and `satisfies`

Prefer letting TS infer, and use `satisfies` to check a value against a type **without widening** it — you keep the precise literal shape *and* get the error on mismatch. Real uses: `RegularBookingService.ts:232` (`} satisfies ReturnTypeCreateBooking`) and `:1448` (`} satisfies EventTypeBrandingData`). With a plain `: T` annotation you'd lose the exact keys.

Drill: state the type of the object literal at `:232` with `satisfies` vs. if it had a `: ReturnTypeCreateBooking` annotation.

## Narrowing, assertion functions, `unknown`

Untrusted input arrives as `unknown` and is narrowed by zod `parse` (`getBookingData.ts:96`), not cast. Assertion functions encode runtime-checked invariants into the type system: `assertNonEmptyArray<T>(arr): asserts arr is [T, ...T[]]` (`RegularBookingService.ts:123-127`) tells the compiler the array is a non-empty tuple afterward, so indexing is safe under `noUncheckedIndexedAccess`.

Pitfall checklist:
- [ ] External input is `unknown` → validated, never `as any`.
- [ ] "Can't be empty here" is an assertion function, not a `!`.
- [ ] Narrowing via zod/type-guards, not casts.

Drill: find where a booking body becomes typed and describe what breaks if it were `data as any`.

## Generics that earn their keep

Real generics relate types across a boundary:
- `withReporting<T extends any[], R>(func, name)` (`sentryWrapper.ts:236`) preserves the wrapped function's arg tuple and return type — wrapping doesn't erase the signature.
- `TaskerCreate = <TaskKey extends keyof TaskPayloads>(type, payload: TaskPayloads[TaskKey], ...)` (`tasker.ts:30-34`) links a task-type string to its payload type, so `create("sendWebhook", ...)` demands the right payload — a type-safe string-keyed map.

Drill: explain why `create("sendSms", { some: "object" })` fails to compile given `tasker.ts:6-27`.

## Zod as the validation contract, aligned to the DB

Zod schemas validate at boundaries and double as types via `z.infer`. This repo keeps the Prisma model and the zod schema aligned using `@zod` annotations directly in `schema.prisma` (e.g. `:867` `bookingResponses`, `:859` `emailSchema`, `:193` `eventTypeBookingFields`) — so the DB shape and the validation shape can't silently drift.

The booking body schema is **dynamic**: built per event type from its `bookingFields` (`RegularBookingService.ts:540-549`), so a custom event type with custom questions gets a matching validator. That's a strong pattern — the contract adapts to configuration.

Drill: open `schema.prisma:156-275` and count `@zod.import` directives; explain what they generate and why it matters for keeping validation honest.

## tRPC end-to-end types

`initTRPC` with superjson (`trpc.ts:7-9`) means the server's input/output types flow to the client with no codegen. After `isAuthed` middleware (`sessionMiddleware.ts:6-23`), `ctx.user` is typed non-null, so handlers skip null checks. A `select` that omits a field the client uses is a compile error, not a runtime surprise.

Sharp edge: superjson lets Dates/BigInts survive tRPC serialization — but the *public* booking POST is plain fetch, so there Dates degrade to strings (`create-booking.ts:8-12`). Same app, two serialization contracts.

Drill: trace the type of a booking's `responses` from `schema.prisma` → zod → the handler that reads it.

## Interview angle (whole file)

- "What does `satisfies` do that an annotation doesn't?" → [08-interview-prep/01](../08-interview-prep/01-js-ts-node-deep-dive.md) Q6.
- "`unknown` vs `any`?" → Q4. "Assertion functions?" → Q5. "Generics in real code?" → Q14.
- "How do you get end-to-end type safety?" → Q8, and [08-interview-prep/03](../08-interview-prep/03-api-and-data-modeling-questions.md).
