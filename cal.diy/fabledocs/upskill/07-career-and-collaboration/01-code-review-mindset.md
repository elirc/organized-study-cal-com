# Code Review Mindset

## The five layers (review in this order)

1. **Does it work?** — does it do what the PR claims? (Run it / read the test.)
2. **Is it correct?** — edge cases, races, error paths, off-by-one (the `checkForConflicts` adjacency class).
3. **Will it stay correct?** — is it tested? Will the next change break it silently?
4. **Does it fit?** — follows repo patterns (`select` over `include`, thin handler/fat service, error-type rules, no barrel imports)?
5. **Is it kind to future maintainers?** — readable, named, commented on *why* not *what* (`AGENTS.md`).

Block only on 1-3 (and security). Layers 4-5 are usually "important" or "optional." A reviewer who blocks a PR over a style nit while missing a race has the priorities backwards.

## Repo-specific review checklist

Correctness & security:
- [ ] Mutations authorize the *specific* resource (not just login); team-role checks are real, not the stub.
- [ ] No `Credential.key`/secret in `select`/response/log.
- [ ] External input validated at the boundary (zod).
- [ ] No check-then-write race on shared resources (slots, holds).
- [ ] Side effects run after commit / via the tasker, not inside the transaction.
- [ ] Outbound HTTP to user URLs: signed, redirect-blocked.

Fit:
- [ ] `select` not `include`; `import type` for types; no barrel imports.
- [ ] `ErrorWithCode`/`HttpError` in services, `TRPCError` only in routers.
- [ ] UI strings via i18n (`en/common.json`).
- [ ] PR under ~500 lines / 10 code files, single responsibility.
- [ ] Conventional commit; draft by default.

Tests:
- [ ] New behavior has a test at the right layer; bug fix has a failing-first test.
- [ ] Time/timezone handled deterministically (UTC or timezone mode).

## Example comments (kind + specific)

Blocking, security:
> This returns the full `credential` row, which includes `key` (a secret). Please `select` only `{ id, appId }`. `AGENTS.md` flags this specifically.

Important, correctness:
> This check-then-`upsert` can race: two concurrent requests both pass the check before either writes. Could we lean on the `SelectedSlots` unique index and make the upsert authoritative, dropping the read?

Optional, fit:
> Small thing — this reads cleaner as an early return (`if (!booking) return null;`), matching the pattern in the rest of this file.

Teaching (not blocking):
> Nice use of the scenario builder here. One idea: `getOrganizer` can set the schedule directly, which would shorten the setup.

## What makes a review kind

Assume good intent, critique the code not the author, explain the *why*, and offer a path forward ("could we…?" not "this is wrong"). Praise genuinely when warranted. The goal is a better PR and a better engineer, not a display of cleverness.

## Interview angle

"Walk me through how you review a PR" → the five layers, block only on correctness/security, teach in the rest, with a concrete example. Practice on [04-code-reading-gym/04-review-katas.md](../04-code-reading-gym/04-review-katas.md) and [08-interview-prep/05](../08-interview-prep/05-debugging-and-code-review-rounds.md).

Drill: review Kata 2 (credential leak) and Kata 5 (cache TTL) from the review katas; write one blocking and one important comment. Self-grade against the "kind + specific" examples above.
