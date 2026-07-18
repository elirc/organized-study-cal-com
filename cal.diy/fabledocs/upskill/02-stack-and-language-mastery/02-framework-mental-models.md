# Framework Mental Models (React + Next)

## React: render vs. commit

React **renders** (calls your components to compute the next UI tree) then **commits** (applies the diff to the DOM). Renders are frequent and should be pure; effects (`useEffect`) run after commit for side effects. State placement determines what re-renders: state high in the tree re-renders everything below.

Where this repo uses it: the Booker (`apps/web/modules/bookings/components/Booker.tsx`) is a large client component fed a wide prop surface (`:55-100`) with derived data memoized (`useMemo`, `:36`). Booking submission is a React Query mutation (`hooks/useBookings.ts:136`), not manual `fetch`+state — so loading/error/success states come from the library.

Sharp edges here:
- **Stale closures** in effects/callbacks that omit a dependency (see [01-language-runtime-model.md](01-language-runtime-model.md) closures).
- **Keys**: lists of slots/days need stable keys or React mis-reconciles.
- **Over-memoization**: `useMemo`/`useCallback` everywhere adds cost; use it where a computation is expensive or referential stability prevents a child-render cascade.

Pitfall checklist:
- [ ] Effect dependency arrays are complete (or you use functional updates/refs deliberately).
- [ ] Derived data is computed in render (or memoized), not stored in state and synced via effects.
- [ ] List keys are stable and unique.

Drill: find a `useMemo`/`useCallback` in the Booker tree and judge whether it's load-bearing (expensive compute or stability) or ceremonial.

Interview angle: [08-interview-prep/02](../08-interview-prep/02-frontend-framework-questions.md).

## State: store vs. server-cache vs. URL

Three homes for state, chosen deliberately here:
- **Server cache** (React Query) for server data — bookings, slots. Cache keys, refetch, mutation invalidation (`useBookings.ts`).
- **Client store** (zustand) for view state — `Booker/store.ts:442-708` holds selected date/time/layout/month.
- **URL** as the source of truth for shareable/deep-linkable state — the store mirrors key fields to query params (`getQueryParam`/`updateQueryParam`).

The insight: the Booker uses the URL as truth so a refresh or shared link restores the exact step. That's a senior-level state decision you can cite.

Drill: name one Booker field that lives only in the store and one mirrored to the URL; explain why the split.

Interview angle: [08-interview-prep/02](../08-interview-prep/02-frontend-framework-questions.md) state question.

## Next.js: App Router + the legacy bridge

`apps/web` is mid-migration: new `app/` routes wrap old Pages-Router data loaders. The public booking page (`app/(booking-page-wrapper)/[user]/[type]/page.tsx:15-33`) builds a legacy context and runs the old `getServerSideProps` via `withAppDirSsr`, then renders a client component. `generateMetadata` runs the same loader for SEO (`:36-70`).

Mental model:
- **Server Components / `page.tsx`** run on the server, fetch data, and pass serializable props down. Permission checks go here, never in `layout.tsx` (`AGENTS.md`).
- **Client Components** (`"use client"`) handle interactivity (the Booker).
- The **bridge** (`withAppDirSsr`, `buildLegacyCtx`) lets hundreds of existing loaders keep working during migration.

Sharp edges: async `params`/`searchParams` must be awaited (Next 15, `page.tsx:19`); data can be fetched in both `page` and `generateMetadata` (Next request memoization dedupes); the serialization boundary turns Dates into strings.

Drill: find another `app/` route using `withAppDirSsr` and confirm the same bridge shape.

Interview angle: [08-interview-prep/02](../08-interview-prep/02-frontend-framework-questions.md) — "how would you migrate Pages Router to App Router?"

## Data fetching layer

Authenticated reads/writes go through **tRPC** (typed, `routers/viewer/*`) consumed via React Query on the client. Public booking submission goes through a **plain fetch** to `/api/book/event` (`create-booking.ts`) because it must accept anonymous, defended traffic. Knowing *why* the app uses two mechanisms (type-safe internal vs. hardened public) is the mid-level insight.

Drill: explain why the booking POST isn't a tRPC mutation.

Interview angle: [08-interview-prep/03](../08-interview-prep/03-api-and-data-modeling-questions.md).
