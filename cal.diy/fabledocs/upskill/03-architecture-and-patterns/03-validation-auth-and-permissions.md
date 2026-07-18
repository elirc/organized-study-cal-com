# Validation, Auth & Permissions

Three distinct concerns people conflate: **validation** (is the input well-formed?), **authentication** (who are you?), **authorization** (may you do this to *this* resource?). Cal.diy has clear examples of each — and one instructive gap.

## Validation layers

1. **Transport pre-checks** (public writes): Turnstile, bot detection, rate limit — `apps/web/pages/api/book/event.ts:20-40`. Not validation of *content*, but of *legitimacy*.
2. **Schema validation** (zod): the booking body is parsed against a schema built from the event type's `bookingFields` (`RegularBookingService.ts:540-549`, `getBookingData.ts:96`). tRPC procedures each have a `*.schema.ts`.
3. **Business-rule validation**: throw-early guards — blocked email, active-booking limits, verification, time-in-bounds (`RegularBookingService.ts:580-651`).

The transferable rule: validate **at the boundary**, once, before use; derive types from the schema; keep the schema aligned to the DB (`@zod` annotations).

## Authentication

NextAuth with Credentials/Google/Email providers (`next-auth-options.ts`). Sessions become a typed `ctx.user` via the tRPC `isAuthed` middleware (`sessionMiddleware.ts:6-23`), which throws `UNAUTHORIZED` when absent. `getServerSession` (`getServerSession.ts:39`) reads the session in API routes.

Note the magic-link expiry mismatch (`next-auth-options.ts` ~362: `maxAge: 10*60*60` vs comment "10 min") — a security-relevant config to resolve (see [risk register](../09-reference/risk-register.md) #5).

## Authorization (the interesting part)

Authorization is **per-resource**, not just "logged in." Two shapes:

- **Object-level ownership** — `eventOwnerProcedure` / `createEventPbacProcedure` load the target event and verify the caller owns it or is an assigned user (`eventTypes/util.ts:72-73`, `151-158`). This is a correct **IDOR** defense: you can't act on an id you don't own.
- **Guard chains in domain code** — cancellation checks host identity before allowing the action (`handleCancelBooking.ts:140-176`).

### The gap: team PBAC is a stub

For **team** events, authorization delegates to `PermissionCheckService.checkPermission` (`eventTypes/util.ts:161-167`), but that class (`:15-20`) returns `true` unconditionally. So role enforcement (ADMIN/OWNER) is bypassed for team events. This is the classic "looks enforced, isn't" bug — the enterprise PBAC lived in removed code, leaving a permissive placeholder. Confirmed; see [risk register](../09-reference/risk-register.md) #1 and [05-key-flows.md Flow 7](../01-codebase-cartography/05-key-flows.md).

The assigned-users validation (`util.ts:178-196`) *does* still run, so a caller can't assign an event to users outside the team — that's the remaining real guard.

## Tenant / user / resource isolation

- Personal events scope by `userId`; team events by `teamId` + membership.
- Queries use `select` to avoid leaking fields; **`Credential.key` must never be selected/returned** (`AGENTS.md`; `schema.prisma:308-311`).
- Cross-tenant access is prevented by loading the resource and checking ownership *before* acting — the load-then-check pattern.

## What a junior misses vs. what a senior checks

| Junior misses | Senior checks |
| --- | --- |
| "It's behind auth, so it's safe" | Does it check *this* resource, or just login? |
| Trusts a procedure named `PbacProcedure` | Opens the implementation — finds the stub |
| Validates in the handler after using input | Validates at the boundary before use |
| Selects whole relations (`include`) | `select` least-privilege; never secrets |
| Authz by email equality is fine | Prefers authenticated id; scrutinizes `email === cancelledBy` (`handleCancelBooking.ts:141`) |

## Pre-merge authorization checklist

- [ ] Every mutation loads the target and verifies ownership/role before acting.
- [ ] Deny by default; team-role checks are real, not stubbed.
- [ ] No `Credential.key` (or other secret) in any `select`/response.
- [ ] Input validated at the boundary with a schema.
- [ ] Public endpoints have rate-limit + bot/spam defenses.

Drill: write the real team-role check to replace `eventTypes/util.ts:161-167` (compare caller's `team.members` role to `fallbackRoles`, deny otherwise). Then write the failing test that proves the current stub is broken. Self-grade — **Strong:** test fails today, passes after your guard, and defaults to deny.

Interview angle: "authn vs authz", "what is IDOR and how do you prevent it", "tell me about a security bug you found" → [08-interview-prep/03](../08-interview-prep/03-api-and-data-modeling-questions.md) and [06 behavioral](../08-interview-prep/06-behavioral-star-stories.md).
