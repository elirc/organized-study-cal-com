# Cal.diy Upskill Curriculum

A training lab built on this repo for one learner: a junior fullstack JS engineer (React/Node/TS CRUD experience) who wants to reach mid-level fast, build senior judgment, and pass interviews for mid-level fullstack roles. Every module teaches two things at once: **where things live in this codebase** and **the transferable pattern you can name in any other repo — and in an interview**.

## What this repo is

Cal.diy is the MIT-licensed, community-driven fork of Cal.com with the enterprise features (Organizations, Insights, Workflows, SSO/SAML) stripped out — see [README.md](../../README.md). It is a **scheduling platform**: hosts define event types with availability rules; visitors book slots on public pages; the system syncs calendars, sends emails/SMS, fires webhooks, and handles payments through an app store of integrations. It is a Yarn-workspaces + Turborepo monorepo: `apps/web` is a Next.js app (App Router pages bridging into legacy `getServerSideProps`), `apps/api/v2` is a NestJS platform API, and ~25 `packages/*` hold the domain logic — most importantly `packages/prisma` (2,851-line schema), `packages/trpc` (authenticated API surface), `packages/features` (booking engine, availability, webhooks, tasker), and `packages/app-store` (integrations). The interesting engineering lives in the booking pipeline (`packages/features/bookings/lib/service/RegularBookingService.ts`, 2,690 lines) and the availability engine (`packages/trpc/server/routers/viewer/slots/util.ts`, 1,435 lines).

This is a *large production codebase*: it will teach you dependency injection, repository/service layering, idempotency, slot-reservation races, webhook signing, and background task queues — the exact vocabulary mid-level interviews probe for.

## How to use it

- **One weekend** — do [00-fast-track.md](00-fast-track.md) only.
- **Two weeks** — fast track, then [01-codebase-cartography/05-key-flows.md](01-codebase-cartography/05-key-flows.md), [03-architecture-and-patterns/05-pattern-catalog.md](03-architecture-and-patterns/05-pattern-catalog.md), and the whole of [08-interview-prep/](08-interview-prep/README.md) via the [two-week cram plan](08-interview-prep/07-two-week-cram-plan.md).
- **Eight weeks** — one module per week in numeric order, doing every drill; ship 2-3 tickets from [06-contribution-practice/01-good-first-tickets.md](06-contribution-practice/01-good-first-tickets.md) along the way.
- **Ongoing contributor** — treat [06-contribution-practice/](06-contribution-practice/README.md) as your backlog and [09-reference/risk-register.md](09-reference/risk-register.md) as your investigation list.

### Recommended paths by profile

| You are | Start | Then |
| --- | --- | --- |
| Brand-new junior | 00-fast-track | 01 → 02 → 04 (drills before theory) |
| Junior who knows React/Node/TS | 00-fast-track | 01/05-key-flows → 03 → 05 → 06 |
| Mid-level, new to this repo | 01-cartography | 03/06-architecture-critique → 06-contribution |
| Senior doing architecture review | 03/06-architecture-critique | 09/risk-register → 05-quality |
| **Interview in two weeks** | [08-interview-prep/07-two-week-cram-plan.md](08-interview-prep/07-two-week-cram-plan.md) | follow it day by day |

## Conventions in these docs

- **File anchors** — every codebase claim cites `path:line-line` (e.g., `apps/web/pages/api/book/event.ts:37-40`). Line numbers were verified against the working tree on 2026-07-11; they will drift as the repo changes — trust the path and the described shape over the exact number.
- **Fake code is labeled** — any snippet not from this repo starts with `// Illustrative fake code: not from this repo`. Unlabeled excerpts are real.
- **Drills and self-grading** — most sections end with a task and a Basic/Solid/Strong rubric. Grade yourself honestly; "Strong" is the mid-level interview bar.
- **`verified` vs `inferred`** — commands marked `verified` were executed while writing these docs; `inferred` come from `package.json`/`agents/commands.md` but were not run here (see [09-reference/verification-log.md](09-reference/verification-log.md)).
- **Suspicions are labeled** — "investigate" / "possible risk" means unproven; "confirmed" means the code was read end to end.

## Learning tracks

1. **[01-codebase-cartography](01-codebase-cartography/README.md)** — the map: system shape, reading order, glossary, tooling, key flows.
2. **[02-stack-and-language-mastery](02-stack-and-language-mastery/README.md)** — TS/Node/React/tRPC mental models anchored to real files.
3. **[03-architecture-and-patterns](03-architecture-and-patterns/README.md)** — boundaries, persistence, authz, async reliability, pattern catalog, critique.
4. **[04-code-reading-gym](04-code-reading-gym/README.md)** — annotation drills, trace tables, fake-code contrasts, review katas.
5. **[05-quality-engineering](05-quality-engineering/README.md)** — testing, debugging, performance, security, observability.
6. **[06-contribution-practice](06-contribution-practice/README.md)** — tickets and projects with realistic blast-radius analysis.
7. **[07-career-and-collaboration](07-career-and-collaboration/README.md)** — review mindset, PRs/RFCs, maintainer communication.
8. **[08-interview-prep](08-interview-prep/README.md)** — 40+ question cards (≥60% repo-anchored), system design from this repo, STAR stories, cram plan.
9. **09-reference** — [command cheatsheet](09-reference/command-cheatsheet.md), [risk register](09-reference/risk-register.md), [rubrics](09-reference/learning-rubrics.md), [verification log](09-reference/verification-log.md).

## The mindset ladder

- **Junior asks:** "How do I make it work?"
- **Mid-level asks:** "Is this the right pattern? What breaks it?"
- **Senior asks:** "What does this commit us to, who pays the cost, and how do we reduce the risk?"

Mid-level interview loops are testing exactly the second and third questions. When this curriculum shows you a slot-reservation race (`reserveSlot.handler.ts:68-103`) or a permission stub that always allows (`eventTypes/util.ts:15-20`), it is handing you an interview answer: a concrete example, a tradeoff, and a failure mode. Collect them.
