# Two-Week Cram Plan

Fourteen days to interview-ready for a mid-level fullstack role, using Cal.diy as your evidence bank. ~2-3 focused hours/day. Checkpoints at day 7 and day 14. Adjust to your gaps.

The through-line: by the end you can answer any mid-level question with a **concrete anchor + tradeoff + failure mode**, whiteboard the scheduling system, run debugging/review rounds aloud, and tell 6 STAR stories.

## Week 1 — build the foundation

**Day 1 — Run it & map it.** Do [00-fast-track.md](../00-fast-track.md) through the two flow traces. Read [01-codebase-cartography/01-system-map.md](../01-codebase-cartography/01-system-map.md). Goal: run the app (or read the seed), draw the write + read paths.

**Day 2 — The write path cold.** Study [Flow 1](../01-codebase-cartography/05-key-flows.md) and [Flow 4](../01-codebase-cartography/05-key-flows.md). Do annotation drills 2, 5, 6 ([04/01](../04-code-reading-gym/01-annotation-drills.md)). Teach-back: booking creation aloud in 3 min.

**Day 3 — The read path & concurrency.** Study Flows 2 and 3. Do drill 4 and debugging scenario 2 ([05/03](../05-quality-engineering/03-systematic-debugging.md)). Be able to explain the reservation TOCTOU.

**Day 4 — Language cards.** [08/01 JS-TS-Node](01-js-ts-node-deep-dive.md), all 14. Answer aloud; re-open anchors you can't cite. Focus Q1-Q6, Q9, Q14.

**Day 5 — Frontend cards.** [08/02 frontend](02-frontend-framework-questions.md), all 12. Then [02-stack/02 framework mental models](../02-stack-and-language-mastery/02-framework-mental-models.md) for depth.

**Day 6 — API & data cards.** [08/03 API/data](03-api-and-data-modeling-questions.md), all 13. Then [03-architecture/02 data model](../03-architecture-and-patterns/02-data-model-and-persistence.md).

**Day 7 — CHECKPOINT + first mock system design.** Do [08/04 system design](04-system-design-from-this-repo.md) end to end, aloud, 25 min. Self-assess against [09/learning-rubrics](../09-reference/learning-rubrics.md).
- Checkpoint bar: trace both hot paths from memory; answer 70% of cards at the mid-level bar; complete the system-design steps 1-5 with at least one anchor each.

## Week 2 — sharpen & rehearse

**Day 8 — Patterns.** [03/05 pattern catalog](../03-architecture-and-patterns/05-pattern-catalog.md), all 16. For each, find the concept in code that *isn't* this repo (mentally). These are recognition reps.

**Day 9 — Debugging rounds.** [08/05](05-debugging-and-code-review-rounds.md) Rounds A, B, C — timed, aloud, recorded. Grade against rubrics. Redo any that scored Basic.

**Day 10 — Review rounds.** [08/05](05-debugging-and-code-review-rounds.md) Rounds D, E, F + two katas from [04/04](../04-code-reading-gym/04-review-katas.md). Practice kind + specific comments.

**Day 11 — STAR stories.** [08/06](06-behavioral-star-stories.md). Write 6 covering conflict/ambiguity/mistake/tradeoff/ownership. Rehearse each under 2 min, ending on impact.

**Day 12 — Second mock system design + a variation.** Redo [08/04](04-system-design-from-this-repo.md) steps 1-5 in 20 min, then variation #4 (guarantee no double-booking) in 10. Then [03/06 architecture critique](../03-architecture-and-patterns/06-architecture-critique.md) for the "critique this design" prompt.

**Day 13 — Security + performance.** [05/05 security checklist](../05-quality-engineering/05-security-checklist.md) and [05/04 performance](../05-quality-engineering/04-performance-thinking.md). Be able to threat-model the booking API and name 3 hotspots with anchors.

**Day 14 — CHECKPOINT + full rehearsal.** Simulate a loop: 1 coding-adjacent explanation (trace a flow), 1 system design (25 min), 1 debugging round, 1 review round, 3 STAR stories. Record; grade against [09/learning-rubrics](../09-reference/learning-rubrics.md).
- Checkpoint bar: every self-assessment box in the rubric ticked; system design done with anchors from memory; debugging/review narrated as a method; 6 STAR stories under 2 min.

## If you have less time

- **One weekend:** Days 1-3 + [08/04 system design](04-system-design-from-this-repo.md) + skim [08/01](01-js-ts-node-deep-dive.md). You'll have the flows and the flagship design.
- **One week:** Week 1 only, plus Day 9 (debugging) and Day 11 (STAR).

## The night before

Re-read [Flow 1](../01-codebase-cartography/05-key-flows.md), the [pattern catalog](../03-architecture-and-patterns/05-pattern-catalog.md) card titles, and your 6 STAR bullets. Don't cram new material — consolidate. Remember the golden rule: **concrete example + tradeoff + failure mode**, every answer.
