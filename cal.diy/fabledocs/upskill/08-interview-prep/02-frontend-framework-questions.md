# Interview Prep: Frontend Framework Questions

Twelve cards on React and Next.js, most anchored to the Booker. Answer with a concrete example + tradeoff + failure mode. Practice each in 90 seconds.

---

## Q1: Explain React's render vs. commit phases.

Round: frontend. Testing: rendering model. Anchor: `Booker.tsx` (large client component).
Junior: "React updates the DOM." Mid adds: render computes the next tree (must be pure — no side effects), commit applies the diff; effects run after commit. Failure mode: doing side effects during render causes bugs/double-runs (esp. StrictMode). Senior: reconciliation uses keys; unstable keys or new object identities cause needless re-renders/remounts.
Follow-up: "what triggers a re-render?" Drill: name one thing the Booker does in render vs. in an effect.

---

## Q2: When is `useMemo`/`useCallback` worth it, and when is it premature?

Round: frontend/perf. Anchor: `Booker.tsx:36` imports both; derived schedule days memoized.
Junior: "caches values." Mid: memoize expensive computes or stabilize identity to prevent child re-render cascades; keyed on deps. Failure mode: wrong deps → stale (stale closure); memoizing cheap values adds cost. Senior: measure first; most `useMemo` is noise; it matters at a real bottleneck or a referential-stability requirement.
Drill: find a memo in the Booker and judge if it's load-bearing.

---

## Q3: What's a stale closure and how do you avoid it?

Round: frontend. Anchor: any effect/callback in `useBookings.ts`.
Junior: unfamiliar. Mid: a callback captures an old state/prop because it wasn't in the dep array; fix with complete deps, functional updates, or refs. Failure mode: a submit handler using a stale `selectedSlot`. Senior: prefer deriving data over storing+syncing via effects, which removes the class of bug.
Drill: describe a plausible stale-closure bug in slot selection.

---

## Q4: How do you manage complex client state? (store vs. server cache vs. URL)

Round: frontend. Anchor: `Booker/store.ts:442-708` (zustand) + query-param sync; `useBookings.ts` (React Query).
Junior: "useState/context." Mid: three homes — React Query for server data, zustand for view state, URL for shareable/deep-linkable state; chosen per need. Failure mode: two sources of truth drifting. Senior: the Booker uses the URL as source of truth so refresh/shared links restore the exact step — a deliberate tradeoff (history writes, hydration care).
Drill: name a field only in the store vs. one mirrored to the URL.

---

## Q5: Server Components vs. Client Components — what goes where?

Round: frontend/Next. Anchor: `[user]/[type]/page.tsx` (server) vs. `Booker.tsx` (`"use client"`).
Junior: "server renders HTML." Mid: server components fetch data and run on the server (no client JS shipped for them), passing serializable props to client components which handle interactivity. Failure mode: importing server-only code (secrets, `node:crypto`) into a client component. Senior: the serialization boundary turns Dates into strings; permission checks go in `page.tsx`, never `layout.tsx` (`AGENTS.md`).
Drill: why can't the webhook signing run in the Booker?

---

## Q6: How would you migrate a Pages Router app to App Router?

Round: Next/architecture. Anchor: `page.tsx:15-19` `withAppDirSsr(getServerSideProps)`.
Junior: "rewrite the pages." Mid: incrementally — a bridge runs existing `getServerSideProps` loaders under new `app/` routes, so you migrate the shell without rewriting every loader. Failure mode: async `params`/`searchParams` not awaited (Next 15); double data-fetch across page + metadata. Senior: request memoization dedupes; migrate leaf-by-leaf behind the adapter, delete it last.
Drill: find another route using `withAppDirSsr`.

---

## Q7: How does data fetching work here, and why two mechanisms?

Round: frontend/API. Anchor: tRPC (`viewer/*`) for authed; plain fetch (`create-booking.ts`) for the public booking POST.
Junior: "fetch/axios." Mid: authed reads/writes use tRPC + React Query (typed, cached, invalidated); the public booking uses a plain hardened API route because it must accept anonymous, rate-limited, bot-checked traffic. Failure mode: trying to force the public booking through tRPC and losing the defenses. Senior: two serialization contracts (superjson for tRPC keeps Dates; fetch degrades them to strings).
Drill: explain why the booking mutation isn't a tRPC procedure.

---

## Q8: How do you handle loading/error/success states cleanly?

Round: frontend. Anchor: `useBookings.ts:136` `useMutation` exposes `isPending`/`isError`/`error`.
Junior: manual booleans. Mid: React Query gives these for free + retries + invalidation; derive UI from them. Failure mode: hand-rolled flags drift out of sync. Senior: optimistic updates + rollback for snappy UX; the Booker exposes `creatingBooking` (`useBookings.ts:429`) derived from the mutation.
Drill: list the states the booking mutation exposes.

---

## Q9: How do you keep a form with dynamic fields type-safe and validated?

Round: frontend/forms. Anchor: booking form uses per-event-type `bookingFields`; validated by a dynamic zod schema (`RegularBookingService.ts:540-549`).
Junior: manual validation. Mid: the form is generated from `bookingFields` and validated against a matching schema on the server; client and server share the shape. Failure mode: client-only validation trusted by the server. Senior: the schema is built per event type so custom questions get real validation — the contract adapts to config.
Drill: trace a custom question from event-type config to server validation.

---

## Q10: Accessibility basics — what would you check on the booking flow?

Round: frontend/a11y. Anchor: Booker date/time pickers.
Junior: "add alt text." Mid: keyboard navigation, focus management in dialogs, accessible names on controls, color contrast, ARIA only where needed. Failure mode: a custom date picker unusable by keyboard/screen reader. Senior: test with the devtools a11y pane and a screen reader; semantic HTML first, ARIA second.
Drill: find one Booker control and check its accessible name.

---

## Q11: How do you prevent unnecessary re-renders from a global store?

Round: frontend/perf. Anchor: `Booker.tsx:39` uses `shallow` from zustand; store selectors.
Junior: unaware. Mid: subscribe to narrow slices with selectors + shallow equality so a component re-renders only when its slice changes. Failure mode: selecting the whole store re-renders on every change. Senior: `createWithEqualityFn`/`shallow` (used here) controls equality; colocate state to limit blast radius.
Drill: find the `shallow` usage and explain what it prevents.

---

## Q12: How do you make client state survive refresh and be shareable?

Round: frontend. Anchor: Booker URL-param sync (`Booker/store.ts` query-param utils).
Junior: localStorage. Mid: URL query params for shareable/deep-linkable state (a booking step in a link); localStorage for device-local prefs; store for ephemeral view state. Failure mode: putting everything in localStorage (not shareable) or all in the URL (noisy history). Senior: the Booker chose URL-as-truth for the booking step precisely so links restore it.
Drill: what breaks if selected time lived only in component state?

---

## Drill routine

Shuffle, 90 seconds each, mid-level bar. The five with the strongest anchors (Q4, Q5, Q6, Q7, Q9) — practice the senior escalation. Cross-links: [01-js-ts-node-deep-dive.md](01-js-ts-node-deep-dive.md), [04-system-design](04-system-design-from-this-repo.md).
