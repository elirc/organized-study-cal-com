# Maintainer Communication

Contributing to an open-source project (or working with a senior team) is a communication skill as much as a coding one. The rule: **ask for help without outsourcing your thinking.**

## Asking a good question

Bad: "How does booking work?" (outsources your reading.)
Good: "I traced booking creation to `RegularBookingService.ts:486`; I see it re-checks availability at `ensureAvailableUsers`. Is the `SelectedSlots` hold meant to be advisory only, or should it prevent the second booking? I ask because `reserveSlot.handler.ts:67-103` looks racy." — shows the work, asks one specific decision.

Template:
```
Context: what I'm trying to do.
What I found: files/lines I read, my current understanding.
What I tried: approaches + results.
The specific question: the one decision I'm blocked on.
```

## Reporting a bug (repro > opinion)

```
## Summary
One line.

## Steps to reproduce
1. ... 2. ... 3. ...

## Expected vs actual
...

## Evidence
Anchors (file:line), logs, a minimal failing test if you can.

## Scope guess (optional)
Where I think it lives and why — clearly labeled as a guess.
```

A minimal failing test is the strongest bug report — for the PBAC stub, the failing-first test *is* the report.

## Proposing a feature

Lead with the problem and user value, not the solution. Offer a small first step. Ask whether the maintainers want it before building it — a large unsolicited PR is often rejected regardless of quality (`AGENTS.md`: no drive-by refactors). Link an RFC for anything cross-cutting.

## Respectful disagreement

- Separate the decision from the person: "I think X because of Y — what am I missing?"
- Bring evidence (an anchor, a benchmark, a test) not just preference.
- Concede when shown wrong, quickly and graciously — it builds trust.
- Disagree-and-commit: once decided, support it even if you'd have chosen differently.

Example: "I still lean toward the unique-index approach for the reservation fix because it removes the read entirely, but I hear the concern about the seat edge case. Happy to go with the transaction if you'd prefer — either fixes the race."

## Responding to review

- Thank the reviewer; address every comment (fix, or explain why not).
- Don't take it personally; the review is about the code.
- If you disagree, say so with reasoning, then defer to the maintainer's call on their project.
- Re-request review after changes with a short summary of what you changed.

## Templates

Ask: use the four-line template above.
Feature: "Problem → who feels it → tiny first step → 'is this something you'd accept?'"
Bug: the repro template above.
Review reply: "Fixed in <commit>. For the authz comment — I kept the email check because <reason>; happy to change if you'd rather."

## Interview angle

"Tell me about a time you disagreed with a teammate" and "how do you ask for help" are standard behavioral prompts. The strong answer shows: you did your homework first, framed it around evidence, and committed to the outcome. See [08-interview-prep/06](../08-interview-prep/06-behavioral-star-stories.md) (conflict/ambiguity stories).

Drill: write the "ask a good question" message for a real thing you don't understand in this repo (e.g. how seats interact with the idempotency key). Self-grade — **Strong:** it shows your reading, names anchors, and asks exactly one decision.
