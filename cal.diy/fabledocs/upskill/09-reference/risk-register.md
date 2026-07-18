# Risk Register

Confidence: **confirmed** = code read end to end; **investigate** = suspicious, not fully traced. This is a teaching artifact, not a security disclosure — verify independently before acting on production systems. Anchors as of 2026-07-11.

| # | Risk | Evidence (anchor) | Impact | Likelihood | Suggested test | Suggested fix | Confidence |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | Team-event PBAC check always passes | `eventTypes/util.ts:15-20` (stub returns `true`); called `:161-167`; wires `delete` `_router.ts:104-109` | Team member of any role may perform ADMIN/OWNER-gated event-type ops | Medium (needs team + session) | Non-admin member calls `eventType.delete` on team event → expect `FORBIDDEN` (fails today) | Enforce role from `team.members` vs `fallbackRoles`; deny by default | confirmed |
| 2 | Slot reservation TOCTOU | `reserveSlot.handler.ts:67-103` (check then upsert, no txn) | Two visitors both get a hold for one slot | High under concurrency | Concurrency/integration test on double reserve | Rely on unique constraint via single upsert; or wrap in txn; clarify intended soft-hold semantics | confirmed |
| 3 | Reserve error masks root cause | `reserveSlot.handler.ts:104-109` (rethrows "Event type not found") | Misleading incident diagnosis | Medium | Assert accurate error on upsert failure | Preserve original error server-side; return safe generic message with 500 | confirmed |
| 4 | Reserve blocks wrong users for team events | `reserveSlot.handler.ts:80-82` `FIXME` (reserves `eventType.users` = creator, not hosts) | Holds don't reflect actual assignees | Medium | Team-event reserve test | Use routed/host member ids like `getSchedule` does | confirmed |
| 5 | Magic-link maxAge/comment mismatch | `next-auth-options.ts` (~362): `maxAge: 10*60*60` vs comment "10 min" | Auth link valid 10h if value is authoritative — larger exposure window | Medium | N/A (config) | Decide intent; align value+comment; likely `10*60` | confirmed |
| 6 | Webhook no-secret unsigned delivery | `sendPayload.ts:296-299` (`"no-secret-provided"`) | Receiver can't authenticate payload | Low-Medium (only if secret unset) | Test warns/blocks on missing secret | Log/flag; document that receivers must reject the sentinel | confirmed |
| 7 | Webhook subscriber URL is an SSRF surface | `sendPayload.ts:312-320` (`fetch` to attacker-influenced URL; `redirect:"manual"` present) | Server-side request to internal hosts | Medium | Test rejecting private/loopback URLs | Allow-list / block private ranges + DNS-rebind guard (redirect already manual) | investigate |
| 8 | Slots cache key = `JSON.stringify(input)` | `slots/util.ts:104-120` | Cache hit-rate depends on property order / added fields | Low (correctness), Medium (perf) | Test key stability across equivalent inputs | Canonicalize key (sorted keys / explicit fields) | confirmed |
| 9 | Duplicate DI module load | `di/containers/AvailableSlots.ts:36-37` | None functionally; wiring smell | Low | Container resolves once | Delete the duplicate line | confirmed |
| 10 | Idempotency key excludes attendee/event type | `booking-idempotency-key.ts:17-20` | Dedupe by (start,end,userId) only; distinct-attendee concurrency not serialized by key alone | Low (by design; seats/RR handle it) | Test concurrent distinct bookings same slot | Document intent; rely on availability re-check for distinct attendees | confirmed |
| 11 | Cancel authz partly by email equality | `handleCancelBooking.ts:140-141` (`user.email === cancelledBy`) | Host determination trusts `cancelledBy` input | Low-Medium | Test host vs non-host cancel | Prefer authenticated id; treat email path as token-scoped only | investigate |
| 12 | 2s availability cache can show a just-taken slot | `slots/util.ts:79,104-120` | Booker briefly sees an unavailable slot | High (by design) | N/A | Graceful conflict UX on booking; acceptable tradeoff | confirmed |
| 13 | Rate limit keyed on hashed IP only | `apps/web/pages/api/book/event.ts:37-40` | Shared-NAT users share a bucket; single IP throttled together | Low | N/A | Consider composite key (IP + eventType) where appropriate | investigate |
| 14 | Unimplemented webhook data fetchers | `webhooks/.../OOOWebhookDataFetcher.ts:21`, `PaymentWebhookDataFetcher.ts:26`, `RecordingWebhookDataFetcher.ts:26` (`TODO ... Phase 1+`) | Some webhook payloads may be incomplete/empty | Low | Test each fetcher returns expected shape | Implement per TODO | investigate |

## How to use this

- Each **confirmed** row is a ready ticket ([06-contribution-practice](../06-contribution-practice/README.md)) and often an interview story ([08-interview-prep/06](../08-interview-prep/06-behavioral-star-stories.md)).
- Each **investigate** row is a code-reading exercise: trace it, then upgrade it to confirmed or drop it. Do not report investigate-level items as bugs.
- Rows 1, 2, 6, 7 are the security-relevant ones — pair with [05-quality-engineering/05-security-checklist.md](../05-quality-engineering/05-security-checklist.md).
