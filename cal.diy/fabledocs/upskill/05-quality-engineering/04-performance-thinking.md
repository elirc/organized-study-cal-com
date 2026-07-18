# Performance Thinking

Rule zero: **measure first.** Every fix below starts by proving the hotspot exists, then addressing root cause. Anchors as of 2026-07-11.

## Performance domains in this app

| Domain | Where it bites | How to find it |
| --- | --- | --- |
| DB queries | N+1, missing index, unbounded `findMany` | Prisma query logging, `EXPLAIN`, count queries per request |
| Server compute | slot generation per request | timing logs (`slots/util.ts` already logs Redis timing) |
| Caching | low hit rate, stale key | Redis metrics, log cache hit/miss |
| External API latency | calendar busy-time fetches | trace spans, timing around `getBusyTimes` |
| Network payload | over-fetching (`include`) | response size, `select` audit |
| React render | re-render storms in the Booker | React Profiler |
| Bundle size | large client bundles | `ANALYZE=1` (Next bundle analyzer) |

## Likely hotspots (with anchors)

1. **Availability computation** (`slots/util.ts`) — composes many subsystems per request; mitigated by the 2s Redis cache (`:79, :104-120`). If the cache hit rate is low (e.g. the `JSON.stringify` key varies, risk register #8), this recomputes constantly. *Find it:* log cache hit/miss ratio.
2. **External calendar fetches** — busy-time reads dominate latency; there's a calendar-subscription sync layer (`packages/features/calendar-subscription`) precisely to avoid live fetches. *Find it:* span timing around busy-time retrieval.
3. **Booking write** — a single transaction (`createBooking.ts:138-145`); the side-effect fan-out is off-path (tasker), which is the right perf call.
4. **Booker renders** — large prop surface (`Booker.tsx:55-100`); watch for re-renders on every keystroke/selection. *Find it:* React Profiler, check what the zustand store subscriptions trigger.
5. **Large client bundle** — an app this size risks heavy first-load JS; `framer-motion` is lazy-loaded (`LazyMotion` in `Booker.tsx:35`), a good sign. *Find it:* `ANALYZE=1 yarn build`.

## How to find each class of problem

- **N+1:** enable Prisma logging, count queries for one request. A loop issuing per-item queries is the tell (contrast the batched `select` in `createBooking.ts:103-118`).
- **Missing index:** `EXPLAIN` a slow query; check it against the `Booking` indexes (`schema.prisma:918-929`) — is your `WHERE` a prefix of an existing composite? If not, justify a new index.
- **Serial async:** grep for `await` inside `for` loops that don't need ordering — candidates for `Promise.all` (see [02-stack/01](../02-stack-and-language-mastery/01-language-runtime-model.md)).
- **Expensive renders:** React Profiler flame chart; look for components re-rendering when their data didn't change (missing memo or over-broad store selector).
- **Oversized bundles:** bundle analyzer; look for a heavy dep imported eagerly that could be dynamic/lazy.
- **Unbounded queries:** `findMany` without `take`/pagination on a growable table (bookings, tasks).

## The senior instinct

Don't optimize what you haven't measured, and prefer *structural* fixes (an index, a batched query, a cache-key fix, moving work off the request) over micro-optimizations (shaving a `map`). The biggest wins here are: cache-key stability, mirroring external busy times instead of live-fetching, and keeping the booking transaction lean with effects off-path — all of which Cal.diy already partly does.

## Interview framing

"How would you make booking-link page loads faster under 10x traffic?" → measure (where's the time: external calendar fetch vs. compute vs. render), then: cache/warm availability, mirror external busy times via subscriptions, ensure the cache key is stable, rate-limit writes (already keyed on hashed IP). Name the anchor for each. See [08-interview-prep/04](../08-interview-prep/04-system-design-from-this-repo.md) variation #3.

Drill: pick the availability path and write the three measurements you'd take before changing anything (cache hit rate, external-fetch p99, compute time). Self-grade — **Strong:** you measure before proposing, and your fix is structural.
