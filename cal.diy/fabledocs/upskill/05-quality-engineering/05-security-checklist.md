# Security Checklist

Mapped to real code. Cal.diy handles untrusted public traffic (bookings), stores integration secrets, and makes outbound calls to user-supplied URLs — so all three of the big web-security families apply. Anchors as of 2026-07-11.

## The map

| Threat | Where it applies | Current state |
| --- | --- | --- |
| **Authorization / IDOR** | event-type & booking mutations | Personal: real ownership check (`eventTypes/util.ts:151-158`). Team: **stub always allows** (`:15-20`) — gap |
| **Authentication** | login, sessions, magic links | NextAuth (`next-auth-options.ts`); magic-link `maxAge` mismatch (~362) — review |
| **Input validation** | booking body, all tRPC inputs | zod at boundary (`getBookingData.ts:96`, `*.schema.ts`) — good |
| **SSRF** | webhook delivery to subscriber URL | `fetch` with `redirect:"manual"` (`sendPayload.ts:319`) — partial; allow-listing unverified |
| **XSS** | user content rendered in UI/emails | `markdownToSafeHTML` (`eventTypes/util.ts:3`) sanitizes; verify all user HTML paths |
| **CSRF** | state-changing requests | NextAuth CSRF + `apps/web/app/api/csrf`; session cookies |
| **Injection (SQL)** | queries | Prisma parameterizes — low risk; avoid raw SQL |
| **Secrets exposure** | `Credential.key` | must never be `select`ed/returned (`AGENTS.md`, `schema.prisma:308-311`) |
| **Rate limiting / abuse** | booking, auth | `checkRateLimitAndThrowError` keyed on hashed IP (`event.ts:37-40`); Turnstile; bot detection |
| **Webhook auth** | outbound payloads | HMAC-SHA256 signature (`sendPayload.ts:296-318`); no-secret fallback receivers must reject |
| **Cookies** | reservation `uid`, sessions | SameSite None/Lax + Secure logic for embeds (`reserveSlot.handler.ts:111-122`) |
| **Uploads / avatars** | avatar routes | `apps/web/app/api/avatar` — verify content-type/size limits |
| **Dependency risk** | many integrations | lockfile pinned; app-store apps expand surface |

## The confirmed gaps (act on these)

1. **Team PBAC bypass** (`eventTypes/util.ts:15-20`) — a team member of any role passes ADMIN/OWNER-gated checks. Highest-priority fix. Failing-first test in [02-writing-tests-here.md](02-writing-tests-here.md) Recipe 4.
2. **Webhook SSRF surface** (`sendPayload.ts:312-320`) — subscriber URL is user-supplied; `redirect:"manual"` helps but private-range/DNS-rebind blocking is unverified. Investigate; add allow-listing.
3. **Magic-link expiry** (`next-auth-options.ts` ~362) — 10h value vs "10 min" comment; resolve deliberately.
4. **Unsigned webhook fallback** (`sendPayload.ts:299`) — `"no-secret-provided"`; ensure receivers reject and consider warning on send.

## Reviewing a booking-related change: quick threat model

- Does it introduce a new query on a user/team resource? → add the ownership/role check (load-then-verify).
- Does it return more fields? → confirm no secret (`Credential.key`) leaks; prefer `select`.
- Does it accept new input? → validate with zod at the boundary.
- Does it make an outbound HTTP call to user-supplied data? → sign it, block redirects, allow-list the host.
- Does it render user content? → sanitize (`markdownToSafeHTML`) or escape.
- Does it add a public endpoint? → rate limit + bot/spam defense.

## Pre-merge security checklist

- [ ] Every mutation authorizes the *specific* resource (not just "logged in"); team-role checks are real.
- [ ] No secret fields in any `select`/response/log (`Credential.key`, tokens).
- [ ] All external input validated at the boundary with a schema.
- [ ] Outbound requests to user URLs are signed, redirect-blocked, and host-allow-listed.
- [ ] User-rendered content is sanitized/escaped.
- [ ] New public endpoints have rate limiting + abuse protection.
- [ ] Cookies set with correct `SameSite`/`Secure`.
- [ ] No secrets in code, logs, or error context (`withReporting` redacts args — verify your new fields are covered).

Interview angle: "walk me through the security considerations of this booking API" → public entry defenses (rate/bot/Turnstile), schema validation, authz (and the PBAC gap you found), secret handling, webhook SSRF+signing. Naming the confirmed gap with an anchor is a strong signal. See [08-interview-prep/03](../08-interview-prep/03-api-and-data-modeling-questions.md) and [risk register](../09-reference/risk-register.md).
