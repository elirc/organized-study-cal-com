# Learning Rubrics

Observable behaviors, not vague traits. Self-assess honestly against this repo. The **Interview-ready** column is the bar for a mid-level fullstack loop.

## Codebase navigation

| Skill | Junior | Mid | Senior | Interview-ready |
| --- | --- | --- | --- | --- |
| Find a flow | Greps for a string | Traces route→service→repo→DB with anchors | Predicts where logic *should* live before looking | Can narrate the booking flow with file names from memory |
| Read a large file | Reads top to bottom | Skims imports as a TOC, jumps to the handler | Identifies the 3 lines that carry the invariant | Explains `RegularBookingService` structure in 2 min |
| Understand boundaries | Sees files | Names layer ownership (route/service/repo) | Spots boundary leaks | Contrasts thin-handler/fat-service with an alternative |

## Backend judgment

| Skill | Junior | Mid | Senior | Interview-ready |
| --- | --- | --- | --- | --- |
| Validation | Adds an `if` | Uses a schema at the boundary, infers types | Keeps schema as single source of truth, aligned to DB | Explains zod + `z.infer` + the Prisma `@zod` alignment |
| Authorization | Checks logged-in | Checks resource ownership (stops IDOR) | Deny-by-default; audits every mutation path | Explains authn-vs-authz with the middleware/stub contrast |
| Concurrency | Unaware | Recognizes a race | Chooses constraint/txn/lock deliberately | Walks the reserve TOCTOU and 3 fixes with costs |
| Idempotency | Unaware | Adds a unique key | Designs the key's inputs; makes consumers idempotent | Explains booking key + at-least-once webhooks |
| Async/reliability | Fire-and-forget | Queues side effects, expects retries | Argues outbox vs. post-commit enqueue | Designs reliable webhook delivery end to end |

## Frontend judgment

| Skill | Junior | Mid | Senior | Interview-ready |
| --- | --- | --- | --- | --- |
| State placement | Everything in component | Store vs. server-cache vs. URL, chosen deliberately | Minimizes state; derives the rest | Explains Booker's URL-as-truth choice |
| Effects/closures | Copies deps | Correct deps; avoids stale closures | Prefers derived data over effects | Diagnoses a stale-closure bug aloud |
| Performance | Adds `useMemo` everywhere | Measures, memoizes what matters | Fixes root cause (renders, data shape) | Judges whether a `useMemo` is load-bearing |

## Quality

| Skill | Junior | Mid | Senior | Interview-ready |
| --- | --- | --- | --- | --- |
| Testing | Tests happy path | Tests validation/permission/edge, isolates time | Tests at the right layer; prevents flake | Writes a failing regression test first |
| Debugging | Adds logs randomly | Reproduce→narrow→hypothesize→fix root | Bisects layers efficiently; fixes cause not symptom | Narrates a debugging round with a regression test at the end |
| Security | Unaware | Knows OWASP basics, checks authz/input | Threat-models a feature | Names IDOR/SSRF/idempotency in this repo with anchors |

## Collaboration

| Skill | Junior | Mid | Senior | Interview-ready |
| --- | --- | --- | --- | --- |
| PRs | Big, vague | Small, single-responsibility, tested, described | Splits by layer/dependency; states risk+rollback | Writes a PR description a maintainer merges fast |
| Review | "LGTM" | Finds correctness + style, kind + specific | Teaches through review; blocks only what matters | Reviews a kata with blocking/important/optional |
| Communication | Outsources thinking | Asks with a repro + attempt | Proposes options with tradeoffs | Tells a STAR story with impact + a metric |

## Self-assessment checklist (tick when true against this repo)

- [ ] I can run, seed, and open the app.
- [ ] I can trace a booking from click to DB row, naming files.
- [ ] I can explain how double-booking is (and isn't) prevented.
- [ ] I found the PBAC stub myself and wrote a failing test for it.
- [ ] I can name three patterns from [03/05-pattern-catalog](../03-architecture-and-patterns/05-pattern-catalog.md) in code that isn't this repo.
- [ ] I shipped (or fully planned) 2+ tickets with tests.
- [ ] I can whiteboard the scheduling system design in 25 minutes with anchors.
- [ ] I have 6+ rehearsed STAR stories under 2 minutes each.

If every box is ticked, you are at the mid-level interview bar for this domain.
