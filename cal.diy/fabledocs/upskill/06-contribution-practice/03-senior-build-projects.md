# Senior Build Projects

Six projects (2 days–4 weeks) a maintainer might actually accept. Each: problem, product value, design checklist, architecture decisions, likely files, migration/test/security/perf plans, rollout/rollback, open questions, stretch. Anchors as of 2026-07-11.

---

## Project 1: Real permission (PBAC) service for teams

**Problem:** team-role authorization is a stub (`eventTypes/util.ts:15-20`); the enterprise PBAC was removed for the fork. **Value:** correct multi-user security — the single most important gap.

Design checklist: define roles/permissions the community edition needs (OWNER/ADMIN/MEMBER × event-type/booking/webhook actions); a `PermissionService` reading `Membership`/`team.members`; deny-by-default; a flag to enforce.

Architecture decisions: keep it minimal (role→permission map), not a full policy engine; enforce at the tRPC procedure boundary (where `createEventPbacProcedure` already sits); make it injectable/testable.

Likely files: `packages/features/pbac/` (new service), `eventTypes/util.ts`, every `createEventPbacProcedure` caller (`_router.ts`), `Membership` model.

Plans — Migration: none (uses existing roles). Test: role-matrix unit + cross-tenant integration. Security: this *is* the security fix. Perf: one membership lookup (already loaded in the middleware). Rollout: flag defaulting to enforce, canary. Rollback: flag off.

Open questions: which actions need finer-than-role permissions? Stretch: extend to bookings and webhooks.

Interview story potential: "I designed and shipped the team authorization model for an open-source scheduling platform." (Flagship security STAR.)

---

## Project 2: Transactional outbox + reliable side-effect delivery

**Problem:** effects enqueued after commit can be lost on crash ([03/04](../03-architecture-and-patterns/04-side-effects-async-and-reliability.md)). **Value:** no silent "booking succeeded, no confirmation."

Design checklist: write task rows inside the booking transaction; a relay dispatches committed-but-unsent tasks; dedupe via `referenceUid`; dead-letter + alert on exhaustion.

Architecture decisions: reuse the `Task` table as the outbox; relay = the existing processor with a "pending dispatch" query; keep at-least-once + idempotent consumers.

Likely files: `createBooking.ts` (enqueue in txn), `tasker/*`, `Task` model, `task-processor.ts`.

Plans — Migration: possibly a task status column. Test: fault-injection (kill after commit → task still dispatches); dead-letter test. Security: n/a. Perf: one extra insert in the txn. Rollout: flag, keep old path. Rollback: flag.

Open questions: relay polling interval vs. latency. Stretch: exactly-once-ish consumer keys per effect.

Interview story potential: "I made booking notifications crash-safe with a transactional outbox." (Reliability senior STAR.)

---

## Project 3: Availability performance — mirror external busy times

**Problem:** live external-calendar fetches dominate slot latency. **Value:** faster, more resilient availability under load.

Design checklist: use the calendar-subscription sync (`packages/features/calendar-subscription`) to mirror busy times locally; slots read the mirror, not live APIs; invalidate on webhook/subscription updates.

Architecture decisions: local busy-time store keyed by credential+range; freshness strategy (subscription push + periodic reconcile); fall back to live fetch on miss.

Likely files: `calendar-subscription/*`, `slots/util.ts` (busy-time source), a busy-time repository.

Plans — Migration: a busy-time cache table. Test: integration on sync + slot correctness; timezone tests. Security: n/a. Perf: measure p99 before/after (the whole point). Rollout: shadow-read compare, then cut over behind flag. Rollback: flag to live fetch.

Open questions: staleness tolerance vs. correctness. Stretch: per-host cache warming for popular links.

Interview story potential: "I cut availability p99 by mirroring external calendar busy times." (Performance senior STAR.)

---

## Project 4: Waitlist for fully-booked slots

**Problem:** no recourse when a slot is full. **Value:** capture demand; auto-book on cancellation.

Design checklist: a `Waitlist` entry (eventType, slot, booker); on cancellation, notify/auto-book the next waiter; expiry; dedupe.

Architecture decisions: hook into the cancellation flow (`handleCancelBooking.ts`) to trigger promotion via the tasker; reuse booking creation for promotion; idempotent promotion.

Likely files: new `Waitlist` model + migration, `handleCancelBooking.ts`, a promotion task, Booker UI.

Plans — Migration: new table (nullable-first, expand). Test: promotion on cancel; no double-promotion (idempotency). Security: authorize waitlist join like booking. Perf: promotion is async. Rollout: per-event-type opt-in flag. Rollback: disable flag; table dormant.

Open questions: fairness (FIFO vs. priority); notification vs. auto-book. Stretch: hold-then-confirm window for the promoted waiter.

Interview story potential: "I designed a waitlist feature end to end — schema, cancellation hook, async promotion, UI." (Full-stack feature STAR.)

---

## Project 5: Extract the booking engine into a testable, documented module

**Problem:** `RegularBookingService.ts` is 2,690 lines with coverage only at the seams. **Value:** maintainability, safer changes.

Design checklist: characterization tests first (lock current behavior), then extract cohesive steps (validation, dedupe, side-effect orchestration) into named, injected collaborators — continuing the existing `handleNewBooking/*` decomposition.

Architecture decisions: no behavior change; each extraction behind green characterization tests; keep the DI interface (`IBookingServiceDependencies`).

Likely files: `service/RegularBookingService.ts`, `handleNewBooking/*`, new focused modules + tests.

Plans — Migration: none. Test: characterization before, unit after each extraction. Security: unchanged (careful not to drop a guard). Perf: unchanged. Rollout: incremental PRs (per `AGENTS.md` size limits). Rollback: per-PR revert.

Open questions: where are the natural seams? Stretch: a documented state machine for booking status transitions.

Interview story potential: "I safely decomposed a 2,700-line service using characterization tests." (Refactoring senior STAR.)

---

## Project 6: Booking analytics dashboard (read model)

**Problem:** no aggregate view of bookings (the enterprise Insights was removed). **Value:** hosts see trends without heavy live queries.

Design checklist: a read model / materialized aggregates (bookings by status/day/event type); a query API; a dashboard UI; incremental update on booking events.

Architecture decisions: don't aggregate live over `Booking` at scale — maintain a denormalized summary updated by a task on booking events; leverage existing indexes for backfill.

Likely files: new aggregate table + migration, a task updating it (tasker), a tRPC query, UI.

Plans — Migration: summary table. Test: aggregate correctness vs. source; backfill idempotency. Security: scope aggregates by owner/team (authz!). Perf: reads hit the summary, not `Booking`. Rollout: backfill then enable. Rollback: drop reads; table dormant.

Open questions: real-time vs. hourly rollup. Stretch: export CSV; per-event-type funnels.

Interview story potential: "I built a denormalized analytics read model to replace expensive live aggregation." (Data-modeling senior STAR — see [08/03](../08-interview-prep/03-api-and-data-modeling-questions.md).)

---

## How to choose

Project 1 (PBAC) is the highest-impact and the best story. Projects 2 and 3 are the strongest *senior-signal* (reliability, performance). Do the design note and a characterization/failing-first test before any code — that discipline is what makes these acceptable to a real maintainer and impressive in an interview.
