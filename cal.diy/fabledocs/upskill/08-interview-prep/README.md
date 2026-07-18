# 08 · Interview Prep

A first-class module. Target: **mid-level fullstack JS interviews.** At least 60% of the question cards are anchored to real Cal.diy code, so you practice answering with concrete evidence instead of memorized definitions — the single biggest differentiator in the room.

## How mid-level fullstack loops are structured

1. **Recruiter/screen** — motivation, basics.
2. **Technical phone screen** — one coding problem + language questions.
3. **Coding round** — practical problem, sometimes in a repo.
4. **System design** — design a product/service, discuss tradeoffs.
5. **Debugging / code review** — find the bug, review the PR.
6. **Behavioral** — STAR stories: conflict, ambiguity, mistake, tradeoff, leadership.

This module has a file for each.

## Files

1. [01-js-ts-node-deep-dive.md](01-js-ts-node-deep-dive.md) — 14 language/runtime cards.
2. [02-frontend-framework-questions.md](02-frontend-framework-questions.md) — React/Next cards.
3. [03-api-and-data-modeling-questions.md](03-api-and-data-modeling-questions.md) — API/schema/validation cards.
4. [04-system-design-from-this-repo.md](04-system-design-from-this-repo.md) — the flagship whiteboard walkthrough.
5. [05-debugging-and-code-review-rounds.md](05-debugging-and-code-review-rounds.md) — timed simulations.
6. [06-behavioral-star-stories.md](06-behavioral-star-stories.md) — STAR worksheets from your repo work.
7. [07-two-week-cram-plan.md](07-two-week-cram-plan.md) — day-by-day plan.

## The golden rule

Answer with a **concrete example, a tradeoff, and a failure mode** — not a definition. "Idempotency means retries are safe" is a junior answer. "This repo derives a booking idempotency key from (start, end, userId) in a Prisma extension and enforces it with a unique constraint, which makes double-submits safe but means distinct attendees on the same slot need seats logic" is a mid/senior answer. Every card below trains that move.

Start with the [two-week cram plan](07-two-week-cram-plan.md) if your interview is soon.
