# 03 · Architecture & Patterns

How Cal.diy is structured, and the reusable patterns inside it. This module turns "I can read the code" into "I can judge the design" — the mid→senior jump interviewers probe.

1. [01-boundaries-and-layers.md](01-boundaries-and-layers.md) — layers, what each owns, real boundary leaks.
2. [02-data-model-and-persistence.md](02-data-model-and-persistence.md) — schema, relationships, indexes, transactions, safe migrations.
3. [03-validation-auth-and-permissions.md](03-validation-auth-and-permissions.md) — validation layers, authn vs authz, IDOR, the PBAC stub.
4. [04-side-effects-async-and-reliability.md](04-side-effects-async-and-reliability.md) — side-effect map, idempotency, retries, outbox, timeouts.
5. [05-pattern-catalog.md](05-pattern-catalog.md) — 16 pattern cards for recognition.
6. [06-architecture-critique.md](06-architecture-critique.md) — strengths, risks, "what I'd change owning this for 3 months" — doubles as system-design interview prep.

Read 05 (pattern catalog) and 06 (critique) even if you skim the rest — they carry the most interview signal.
