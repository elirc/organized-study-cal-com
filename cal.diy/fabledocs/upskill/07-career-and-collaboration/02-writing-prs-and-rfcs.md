# Writing PRs & RFCs

## PR description template (tuned to this repo)

```markdown
## What
One or two sentences: the change and its user/maintainer value.

## Why
The problem. Link the issue/ticket. If it's a bug, the root cause.

## How
Key implementation decisions. What pattern it follows (e.g. "thin handler → RegularBookingService", "select over include"). What you deliberately did NOT do.

## Testing
- [ ] `yarn type-check:ci --force`
- [ ] `yarn test <file>` (name them)
- [ ] Integration/timezone/E2E if relevant
- What layer, what cases (happy / validation / permission / edge).

## Risks & rollback
Blast radius (which entities/flows). How to roll back (revert / flag). Any migration and its expand/contract steps.

## Follow-ups
What's intentionally deferred (link to a ticket).
```

Cal.diy specifics: draft by default; conventional-commit title (`fix:`, `feat:`, `refactor:`); keep under ~500 lines / 10 code files (docs/lockfiles/generated excluded); split by layer if larger.

## Good vs. weak PR

Weak: "Fix booking bug" + 800-line diff touching schema, service, and UI, no tests. A maintainer can't review it.

Good: `fix: prevent unsigned webhook delivery` — 90 lines, one concern, a failing-first test, a "why" (receivers can't authenticate), a rollback (flag), and a linked follow-up for backfilling existing secretless webhooks.

## Commit messages

Conventional commits, imperative, explain *why* in the body when non-obvious:
```
fix: make slot reservation authoritative via unique index

The previous check-then-upsert allowed two concurrent reservations to
both pass the existence check. Rely on the SelectedSlots unique index so
the upsert itself is the source of truth. Documents holds as advisory.
```

## When to write an RFC

Write one before: schema changes with wide blast radius, new cross-cutting patterns (a permission service, an outbox), anything that changes a public contract or is >1 week of work. Skip it for a scoped bug fix.

## RFC template (tailored)

```markdown
# RFC: <title>

## Problem
What's broken/missing and who feels it. Evidence/anchors.

## Proposal
The design. Diagrams if they help. New invariants.

## Data model / API changes
Schema deltas (expand/contract plan). New endpoints/procedures. zod/type impacts.

## Alternatives considered
At least two, with why you rejected them.

## Risk & blast radius
Which flows/entities. Failure modes. Security implications.

## Rollout & rollback
Flags, canary, migration order, revert path.

## Testing plan
Layers and key cases; failing-first / characterization tests.

## Open questions
The decisions you want feedback on.
```

## Worked example: the PBAC RFC (sketch)

- **Problem:** team-role authorization is a stub (`eventTypes/util.ts:15-20`); any member passes ADMIN/OWNER checks.
- **Proposal:** a minimal `PermissionService` reading `team.members` roles vs. `fallbackRoles`, enforced at the tRPC procedure boundary, deny-by-default.
- **Alternatives:** (a) full policy engine — too heavy for community edition; (b) inline role checks per procedure — duplicated, error-prone.
- **Risk:** could over-restrict; mitigate with role-matrix tests + enforce flag.
- **Rollout:** flag defaulting to enforce, canary. **Rollback:** flag off.
- **Tests:** failing-first non-admin delete; per-role matrix; cross-tenant integration.

That RFC is approvable on its own — the bar to aim for.

## Interview angle

"How do you communicate a big technical change?" → RFC first, small PRs after, each with why/testing/risk/rollback. Show the PBAC example. See [08-interview-prep/06](../08-interview-prep/06-behavioral-star-stories.md).

Drill: write the full PR description for [good-first ticket 2](../06-contribution-practice/01-good-first-tickets.md) (reserve error message). Self-grade — **Strong:** a reviewer needs no follow-up questions.
