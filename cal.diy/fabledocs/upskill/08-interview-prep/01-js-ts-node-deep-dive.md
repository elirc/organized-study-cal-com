# Interview Prep: JS / TS / Node Deep Dive

Fourteen question cards. Most are anchored to real code so you answer with evidence, not trivia. For each, the mid-level answer adds a tradeoff + concrete example + failure mode; the senior answer adds an invariant/alternative/when-the-textbook-answer-is-wrong. Practice each aloud in 90 seconds.

Interviewers phrase these many ways; the "What it's really testing" line tells you the actual target.

---

## Q1: Walk me through the event loop. Where would a slow synchronous operation hurt this kind of app?

Round: JS/TS deep-dive. What it's really testing: do you understand non-blocking I/O vs. CPU-blocking.

Repo anchor: the booking handler awaits many async I/O calls in sequence (`RegularBookingService.ts:523-629`) and the tasker runs handlers concurrently (`task-processor.ts:20-37`).

Junior answer: "The event loop runs callbacks; `await` pauses." Mid-level adds: I/O (DB, `fetch`) is offloaded and the loop stays free, but a CPU-bound loop (e.g. crunching a huge availability computation synchronously) blocks *every* request on that Node process — Node is single-threaded for JS. Failure mode: a synchronous JSON.parse of a massive payload or an unbounded loop stalls all concurrent bookings. Senior adds: microtasks (promise callbacks) drain before the next macrotask, so a promise chain can starve timers; the fix for CPU work is a worker thread or moving it off the request (Cal.diy pushes heavy/slow effects to the tasker queue precisely so the request loop stays responsive).

Likely follow-ups: "microtask vs. macrotask ordering?" / "how does this differ in the browser?"

Practice drill: say why availability caching (2s Redis TTL, `slots/util.ts:79`) is about latency, not about unblocking the loop.

---

## Q2: `Promise.all` vs `Promise.allSettled` — when does the choice matter? Find both in a codebase.

Round: JS/TS deep-dive. What it's really testing: failure semantics of concurrent work.

Repo anchor: `reserveSlot.handler.ts:79` uses `Promise.all` to upsert holds for all users; `task-processor.ts:37` uses `Promise.allSettled` to run a batch of tasks.

Junior: "`all` waits for all promises." Mid-level adds the real difference: `all` **rejects on the first failure** (and abandons the others' results), while `allSettled` **always resolves** with per-item status. So the tasker uses `allSettled` (line 37) because one failing task must not abort the whole batch — each task retries independently. Reserve uses `all` (line 79) because if any user's hold fails, the whole reservation should fail. Failure mode: using `all` for a batch job means one bad item kills the batch; using `allSettled` where you *need* atomicity hides failures. Senior adds: `all` doesn't cancel in-flight promises on rejection (JS has no cancellation) — the other upserts still run; and `allSettled` shifts error handling to you (you must inspect `status === "rejected"`, which `task-processor.ts:38` does).

Follow-up: "what about `Promise.race`/`any`?"

Drill: argue why swapping the two usages would be a bug in each case.

---

## Q3: What is a closure, and where is a stale closure a bug risk here?

Round: JS/TS deep-dive / frontend. What it's really testing: closures + React effect pitfalls.

Repo anchor: `withSlotsCache` (`slots/util.ts:104-120`) returns a function closing over `redisClient` and `func`; React hooks like `useBookings.ts` close over props/state.

Junior: "a function that remembers its scope." Mid-level adds a concrete use: `withSlotsCache` captures `redisClient` once and reuses it — intended. The *bug* version is React: an effect/callback that closes over a stale `state`/`prop` because its dependency array omitted it, so it reads an old value. Failure mode: a booking form callback that captured the first-render `selectedSlot`. Senior adds: the fix is correct dependency arrays or functional updates / refs; and that the *same* mechanism (closure captures the value at creation time) is desirable in a HOF like `withSlotsCache` and dangerous in a re-rendering component.

Drill: explain why a module-level singleton container (`di/containers/AvailableSlots.ts:22`) closing over `container` is safe but a component closing over `props` may not be.

---

## Q4: `unknown` vs `any` vs `as any` — what's the rule here and why?

Round: JS/TS deep-dive. What it's really testing: type-safety discipline.

Repo anchor: `AGENTS.md` bans `as any` outright ("use proper type-safe solutions"). Zod parsing turns `unknown` input into typed data (`getBookingData.ts:96`).

Junior: "`any` disables type checking." Mid-level adds: `unknown` is the *safe* top type — you can hold anything but must narrow before use; `any` is contagious and silently propagates, defeating the compiler. The repo's convention is: untrusted input arrives as `unknown` and is narrowed by zod `parse`, not cast. Failure mode: `const x = data as any; x.foo.bar` compiles and crashes at runtime. Senior adds: `as any` also disables *future* refactors' safety (renames won't catch the cast site); prefer `unknown` + a type guard, `satisfies`, or a zod schema. When the textbook is wrong: at genuine library boundaries with no types, a *narrow, commented* `as` on a specific shape beats `any` — but never `as any`.

Drill: find a place input becomes typed via zod and describe what would break if it were `as any` instead.

---

## Q5: Explain TypeScript assertion functions. Show one and say what invariant it encodes.

Round: JS/TS deep-dive. What it's really testing: advanced narrowing.

Repo anchor: `RegularBookingService.ts:123-127` — `function assertNonEmptyArray<T>(arr: T[]): asserts arr is [T, ...T[]]`.

Junior: usually hasn't seen `asserts`. Mid-level: an assertion function throws if a condition fails and, via `asserts arr is [T, ...T[]]`, tells the compiler that *after the call* the array is a non-empty tuple — so subsequent `arr[0]` is typed as `T`, not `T | undefined`. The invariant: "past this line, the array has at least one element." Failure mode without it: `noUncheckedIndexedAccess` makes `arr[0]` possibly-undefined, forcing null checks or an unsafe `!`. Senior adds: assertion functions encode runtime-checked invariants into the type system (like zod does at boundaries); overusing them to paper over "shouldn't happen" cases can hide real bugs — they should represent genuine guarantees.

Drill: rewrite a `arr[0]!` somewhere as an assertion function and explain the improvement.

---

## Q6: What does `satisfies` do that a type annotation doesn't? Find a real use.

Round: JS/TS deep-dive. What it's really testing: modern TS ergonomics.

Repo anchor: `RegularBookingService.ts:232` `} satisfies ReturnTypeCreateBooking;` and `:1448` `} satisfies EventTypeBrandingData)`.

Junior: unfamiliar. Mid-level: `satisfies` checks a value conforms to a type **without widening** the value's inferred type — you get both the compile-time guarantee *and* the precise literal types. With `const x: T = {...}`, `x`'s type becomes `T` (you lose the specific keys/literals); with `const x = {...} satisfies T`, you keep the exact shape while still erroring on mismatch. Failure mode `satisfies` prevents: forgetting a required field in a big object literal (the booking data object) is caught, while still letting you index specific known keys afterward. Senior adds: use `satisfies` for config/return objects where you want literal inference; use a plain annotation when you deliberately want to widen to the interface.

Drill: explain what type `newBookingData` has with `satisfies` vs. if it had a `: ReturnTypeCreateBooking` annotation.

---

## Q7: `import type` — why does this codebase insist on it?

Round: JS/TS deep-dive / tooling. What it's really testing: modules, build correctness.

Repo anchor: `AGENTS.md` ("Use `import type { X }`"); pervasive, e.g. `RegularBookingService.ts:18-23,26,30`.

Junior: "it imports a type." Mid-level: `import type` is erased at compile time, so it (a) can't cause a runtime import/side-effect, (b) avoids circular-dependency runtime cycles between modules that only share types, and (c) helps bundlers/`isolatedModules` know the import is type-only. Failure mode it prevents: importing a value from a module for its type alone, pulling that module's side effects (and its transitive deps) into the bundle. Senior adds: with `isolatedModules`/`verbatimModuleSyntax`, mixing type and value imports can break single-file transpilation; separating them is required, not stylistic. Also relevant to the repo's "no barrel imports" rule — barrels + value imports create fat dependency graphs.

Drill: find two type-only imports and one value import in the same file; explain the runtime difference.

---

## Q8: How does this repo get end-to-end type safety from DB to client?

Round: JS/TS deep-dive / API. What it's really testing: the TS type-flow story.

Repo anchor: Prisma models (`schema.prisma`) → zod schemas (with `@zod` annotations, e.g. `schema.prisma:867`) → tRPC procedures with typed `ctx.user` (`sessionMiddleware.ts:6-23`) → inferred client types.

Junior: "TypeScript everywhere." Mid-level: Prisma generates types from the schema; tRPC infers input/output types so the client gets them without codegen; zod validates at the boundary and `z.infer` reuses the schema as the type. One source of truth per layer. Failure mode: casting breaks the chain; a `select` that omits a field the client expects is a type error, not a runtime surprise. Senior adds: the weak link is any place types are asserted rather than derived (banned `as any`), and the Prisma↔zod boundary where the schema annotations must be kept in sync — this repo automates that with the `@zod` directives so the DB shape and validation shape can't silently diverge.

Drill: trace the type of a booking's `responses` field from schema → zod → handler.

---

## Q9: Node error handling — how does this codebase structure errors, and why two error types?

Round: JS/TS / Node. What it's really testing: error taxonomy and boundaries.

Repo anchor: `AGENTS.md`: "Use `ErrorWithCode` in services/repositories/utilities; use `TRPCError` only in tRPC routers." See `HttpError` throws in `RegularBookingService.ts:589,615,641` and `TRPCError` in `sessionMiddleware.ts:16`.

Junior: "throw an Error." Mid-level: errors are typed by *layer* — domain code throws `ErrorWithCode`/`HttpError` (transport-agnostic), tRPC routers throw `TRPCError` (maps to tRPC error codes), and there's a conversion boundary (`errorConversionMiddleware.ts`, `getServerErrorFromUnknown`) that translates unknown errors into HTTP responses. Failure mode: throwing a `TRPCError` from deep service code couples business logic to the transport; throwing raw strings loses status codes. Senior adds: the pattern is "throw rich, typed errors at the point of failure; translate to transport format at the edge" — one translation boundary, so a service is reusable across REST, tRPC, and the NestJS API. `withReporting` (`sentryWrapper.ts:236`) adds observability by capturing+rethrowing without swallowing.

Follow-up: "what's wrong with `catch {}`?" (swallowing — see `reserveSlot.handler.ts:104-109` which catches and rethrows a *misleading* message).

Drill: find the boundary that turns a thrown `HttpError` into an HTTP status.

---

## Q10: What's the risk in `catch (e) {}` and where does this repo do it questionably?

Round: JS/TS / code review. What it's really testing: error swallowing awareness.

Repo anchor: `reserveSlot.handler.ts:104-109` catches any upsert failure and throws `"Event type not found"` (misleading); `validateRescheduleRestrictions` (`RegularBookingService.ts:476-483`) deliberately swallows non-`HttpError` errors to defer them.

Junior: "you should log it." Mid-level: swallowing hides root causes and produces misleading diagnostics — the reserve handler reports "not found" for what might be a DB write error, sending you debugging the wrong layer. Failure mode: an on-call engineer chases a phantom missing event type. Senior adds: sometimes deliberate swallowing is correct (the reschedule validator at 476-483 intentionally lets "booking not found" fall through to be handled later, and *only* rethrows `HttpError`) — the discipline is: swallow *specific, expected* errors with a comment explaining why, never a bare `catch {}` over everything.

Drill: rewrite the reserve `catch` to preserve the original error while keeping a user-safe message.

---

## Q11: Explain `async`/`await` vs raw promises, and serial vs parallel awaits. Find a serial chain that could be parallel.

Round: JS/TS deep-dive. What it's really testing: latency awareness.

Repo anchor: `RegularBookingService.ts:523-549` awaits event type, then reschedule validation, then body parse — some sequential by dependency; `reserveSlot.handler.ts:79` parallelizes independent upserts with `Promise.all`.

Junior: "await waits." Mid-level: sequential awaits add up latency; independent awaits should run with `Promise.all`. In the booking handler, some steps *must* be serial (you need the event type before you can build its schema), but truly independent I/O (e.g. fetching webhooks + fetching translations) can be parallelized. Failure mode: awaiting in a `for` loop over users instead of `Promise.all` (N sequential round-trips). Senior adds: parallelizing also changes error semantics (Q2) and can overwhelm a downstream (need concurrency limits); and that `await` in a loop is sometimes *intentional* for rate-limited external APIs.

Drill: identify one serial pair in the booking handler that is serial by necessity and one that isn't.

---

## Q12: What is memoization/`useMemo` for, and when is it premature? Point to real UI.

Round: frontend / JS. What it's really testing: render performance judgment.

Repo anchor: `Booker.tsx:36` imports `useMemo`/`useRef`; `Booker.tsx` and slot hooks recompute derived slot data.

Junior: "it caches a value." Mid-level: `useMemo` caches an expensive computation between renders keyed on deps; `useCallback` stabilizes function identity so memoized children don't re-render. In the Booker, memoizing the derived non-empty schedule days avoids recomputing on every keystroke. Failure mode: memoizing cheap values adds overhead and complexity; wrong deps cause stale results (Q3). Senior adds: measure first — most `useMemo` is unnecessary; it matters when (a) the computation is genuinely expensive or (b) referential stability is required to prevent a cascade of child re-renders. Overusing it is a real anti-pattern.

Drill: find a `useMemo`/`useCallback` in the Booker and judge whether it's load-bearing.

---

## Q13: Node vs. edge vs. browser — what constrains where code runs here?

Round: Node / architecture. What it's really testing: runtime boundaries.

Repo anchor: `createHmac` from `node:crypto` (`sendPayload.ts:1`) is Node-only; `process` imports (`slots/util.ts:1`, `event.ts:1`) are server; the Booker (`"use client"`, `Booker/store.ts:1`) is browser.

Junior: "it's all JavaScript." Mid-level: server code can use `node:*` modules, secrets, and the DB; client code cannot (no `node:crypto`, no DB, no secrets — they'd leak). Webhook signing *must* be server-side because it needs the secret and `createHmac`. Failure mode: importing a server-only module into a client component bundles it (or fails), and importing secret env into client code exposes it. Senior adds: Next.js enforces this with `"use client"`/server components and `NEXT_PUBLIC_` env prefixing; the App Router page (`[user]/[type]/page.tsx`) runs on the server and passes serializable props to the client Booker — the serialization boundary is another constraint (Dates become strings, see `create-booking.ts:8-12`).

Drill: explain why `createWebhookSignature` cannot move to the client.

---

## Q14: How do generics show up in real code here (not toy examples)?

Round: JS/TS deep-dive. What it's really testing: practical generics.

Repo anchor: `withReporting<T extends any[], R>(func, name)` (`sentryWrapper.ts:236`) preserves a function's arg tuple and return type; `TaskerCreate = <TaskKey extends keyof TaskPayloads>(type, payload: TaskPayloads[TaskKey], ...)` (`tasker.ts:30-34`) links a task type string to its payload type; `assertNonEmptyArray<T>` (Q5).

Junior: "generics are like templates." Mid-level: real generics preserve or relate types across a boundary. `withReporting` is generic so wrapping a function doesn't lose its signature — callers still get typed args/return. The tasker's `TaskKey extends keyof TaskPayloads` gives a *discriminated* API: `create("sendWebhook", ...)` requires a `string` payload, `create("createCRMEvent", ...)` requires that task's schema type — mismatches are compile errors. Failure mode: without the generic, `withReporting` would return `Function`/`any` and erase types; the tasker would accept any payload for any type. Senior adds: constrained generics (`extends keyof`) are how you build type-safe "string key → typed value" maps — a very common interview-worthy pattern (typed event emitters, typed config).

Drill: explain why `create("sendSms", { some: "object" })` fails to compile given `tasker.ts:6-27`.

---

## How to drill this file

Shuffle the 14 cards. For each, set a 90-second timer and answer aloud at the mid-level bar (tradeoff + example + failure mode). Any card where you can't cite the anchor from memory, re-open the file. Then take the five cards with confirmed-bug anchors (Q2, Q5, Q9, Q10, Q14) and practice the senior escalation — because "here's where the standard answer breaks in real code" is what separates mid from senior in the room. Cross-links: [02-frontend-framework-questions.md](02-frontend-framework-questions.md), [03-api-and-data-modeling-questions.md](03-api-and-data-modeling-questions.md).
