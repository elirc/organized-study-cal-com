# Review Katas

Nine fake PRs to review. All diffs are illustrative (labeled), modeled on real files. For each: review it, categorize findings **Blocking / Important / Optional**, and write one kind, specific comment. Then compare with the expected findings. This is exactly a code-review interview round.

Review principle (from [07/01](../07-career-and-collaboration/01-code-review-mindset.md)): does it work → is it correct → will it stay correct → does it fit → is it kind to future maintainers.

---

## Kata 1: "Speed up slot reservation"

Author intent: remove the "unnecessary" existence check.
```ts
// Illustrative fake diff — resembles reserveSlot.handler.ts
- const reservedBySomeoneElse = await slotsRepo.findReservedByOthers({ slot, eventTypeId, uid });
- if (shouldReserveSlot && !reservedBySomeoneElse) { await upsertHolds(); }
+ if (shouldReserveSlot) { await upsertHolds(); } // upsert is idempotent anyway
```
Files it resembles: `reserveSlot.handler.ts:67-103`.

Expected findings — **Important:** removing the check changes behavior (it was the "someone else holds this" signal); but note the *original* is already a TOCTOU race, so the real fix is neither — it's an authoritative upsert on the unique index. **Optional:** the comment "idempotent anyway" is misleading given the `uid` is in the unique key. Good comment:
> The existence check isn't just an optimization — it's how we tell a visitor the slot is held by someone else. That said, the original check-then-upsert is racy; could we make the upsert authoritative via the `SelectedSlots` unique index and drop the read entirely? That fixes both the race and your perf concern.

---

## Kata 2: "Return richer credential info to the client"

```ts
// Illustrative fake diff
- select: { id: true, appId: true }
+ include: { app: true }, // also pulls `key`
```
Resembles: any credential query; `AGENTS.md`.

Expected — **Blocking:** `include` pulls `key` (a secret) into the response; never expose it. Good comment:
> This will serialize `credential.key` to the client, which is a secret. Please keep an explicit `select` of only the fields the UI needs. `AGENTS.md` calls this out specifically.

---

## Kata 3: "Add email confirmation to booking creation"

```ts
// Illustrative fake diff — resembles createBooking.ts saveBooking
  return prisma.$transaction(async (tx) => {
    const booking = await tx.booking.create(createBookingObj);
+   await sendConfirmationEmail(booking); // notify the attendee
    return booking;
  });
```
Resembles: `createBooking.ts:138-145`.

Expected — **Blocking:** a slow/failing email now rolls back the booking (side effect inside the transaction). Move it after commit, ideally via the tasker. Good comment:
> Sending email inside the transaction ties booking success to SMTP availability — a timeout would roll back a valid booking. Let's commit first and enqueue the email through the tasker (`tasker.create(...)`), which also gives us retries.

---

## Kata 4: "Simplify auth for team events"

```ts
// Illustrative fake diff — resembles eventTypes/util.ts
- const hasPermission = await permissionCheckService.checkPermission({ userId, teamId, permission, fallbackRoles });
- if (!hasPermission) throw new TRPCError({ code: "FORBIDDEN" });
+ // team members are trusted; skip the check
```
Resembles: `eventTypes/util.ts:159-167`.

Expected — **Blocking:** removes authorization entirely for team events (and the existing check is already a stub — so this makes a latent bug explicit). Good comment:
> This drops object-level authorization for team events. Even though `checkPermission` is currently a stub returning `true` (a separate bug we should fix), the intent is an ADMIN/OWNER gate — we can't remove it. Let's instead implement the real role check against `team.members`.

---

## Kata 5: "Cache slots harder"

```ts
// Illustrative fake diff — resembles slots/util.ts
- const DEFAULT_SLOTS_CACHE_TTL = 2000;
+ const DEFAULT_SLOTS_CACHE_TTL = 60000; // 1 minute, fewer DB hits
```
Resembles: `slots/util.ts:79`.

Expected — **Important:** a 60s TTL means a booked slot can show as "available" for up to a minute, worsening the collision UX. **Optional:** if we want this, pair it with cache invalidation on new bookings. Good comment:
> This trades correctness for load: with a 60s TTL a just-booked slot stays "available" much longer, so more users hit the "no longer available" path at submit. If we need fewer DB hits, can we invalidate the cache on booking creation instead of widening the window?

---

## Kata 6: "Fix flaky timezone test by removing TZ"

```ts
// Illustrative fake diff
- TZ=UTC vitest run
+ vitest run // let it use the local timezone
```
Resembles: `package.json` `test`.

Expected — **Blocking:** forcing UTC is what makes tests deterministic across machines/CI; removing it makes results host-dependent. The flake is a real bug to find, not to hide. Good comment:
> Dropping `TZ=UTC` makes tests pass/fail based on the runner's timezone — that's non-determinism, not a fix. Let's find why the test is timezone-sensitive; if it legitimately tests DST behavior, it belongs in the `VITEST_MODE=timezone` suite with an explicit `TZ`.

---

## Kata 7: "Add a booking without an idempotency key"

```ts
// Illustrative fake diff
+ await prisma.booking.create({ data: { ...data, idempotencyKey: crypto.randomUUID() } });
```
Resembles: `createBooking.ts` + `booking-idempotency-key.ts`.

Expected — **Important:** a random key defeats idempotency (every retry gets a new key → duplicates possible); the extension already computes a deterministic key, so setting a random one may conflict with or override intended behavior. Good comment:
> A random `idempotencyKey` means retries won't collide, so we lose double-submit protection. The Prisma extension already derives a deterministic key from (start, end, userId) on create — we should let it, and not pass a random value here.

---

## Kata 8: "Follow webhook redirects"

```ts
// Illustrative fake diff — resembles sendPayload.ts
- redirect: "manual",
+ redirect: "follow", // some subscribers use redirects
```
Resembles: `sendPayload.ts:319`.

Expected — **Important (security):** following redirects on an attacker-influenced subscriber URL widens the SSRF surface and can bounce the signed payload to an unintended host. Good comment:
> `redirect: "manual"` is deliberate SSRF hardening — the subscriber URL is user-supplied, and following redirects could send the signed payload somewhere unexpected. If a subscriber needs a redirect, they should register the final URL. Let's keep manual.

---

## Kata 9: "Refactor: pull conflict check into a one-liner"

```ts
// Illustrative fake diff — resembles checkForConflicts.ts
- for (const busyTime of sortedBusyTimes) { if (busyTime.start >= slotEnd) break; if (busyTime.end <= slotStart) continue; return true; }
- return false;
+ return sortedBusyTimes.some(b => b.start < slotEnd && b.end > slotStart);
```
Resembles: `checkForConflicts.ts:39-49`.

Expected — **Optional (correctness-preserving? verify):** the `.some` version is actually correct and cleaner *for overlap*, but it loses the early `break` optimization (sorted array lets you stop early). **Important** if arrays are large. Good comment:
> Nice simplification and it preserves the half-open overlap semantics. One tradeoff: the original `break` exploits the sorted order to stop early, which `.some` won't. For typical small busy arrays that's fine — worth a comment noting we're trading the early-exit for readability.

---

## How to grade yourself

For each kata: did you catch the Blocking issue? Did you avoid flagging style as blocking? Was your comment specific (cites the risk) and kind (assumes good intent, offers a path)? A strong reviewer blocks only what must be blocked and teaches in the rest. Take Katas 2, 3, 4, 8 into [08-interview-prep/05](../08-interview-prep/05-debugging-and-code-review-rounds.md) as timed rounds.
