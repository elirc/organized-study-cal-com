# Fake-Code Contrasts

Ten bad-vs-better pairs, each tied to a real pattern in this repo. All snippets are labeled illustrative — they are NOT copied from the codebase. The point is to recognize the novice instinct and the better shape, then find where the repo does it right.

---

## 1: Coupling UI shape to DB shape

```ts
// Illustrative fake code: not from this repo
// BAD — sends the raw Prisma row to the client
return res.json(await prisma.credential.findUnique({ where: { id } })); // leaks `key`!
```
```ts
// Illustrative fake code: not from this repo
// BETTER — select only safe fields
return res.json(
  await prisma.credential.findUnique({ where: { id }, select: { id: true, appId: true } })
);
```
Real repo: `reserveSlot.handler.ts:31-33` selects a minimal shape; `AGENTS.md` bans exposing `Credential.key`. Novice instinct: return the whole row. Better: `select` least-privilege.

---

## 2: Missing permission filter (IDOR)

```ts
// Illustrative fake code: not from this repo
// BAD — any logged-in user can delete any event type by id
async function deleteEventType(id: number) { return prisma.eventType.delete({ where: { id } }); }
```
```ts
// Illustrative fake code: not from this repo
// BETTER — load then verify ownership before acting
const ev = await prisma.eventType.findUnique({ where: { id }, select: { userId: true } });
if (ev?.userId !== ctx.user.id) throw new TRPCError({ code: "FORBIDDEN" });
```
Real repo: `eventTypes/util.ts:151-158` does the personal-event ownership check. (And the team branch's stub, `:15-20`, is the cautionary tale — it *skips* this.)

---

## 3: Check-then-write race (TOCTOU)

```ts
// Illustrative fake code: not from this repo
// BAD — two requests both pass the check, both insert
if (!(await slotTaken(slot))) await prisma.hold.create({ data: slot });
```
```ts
// Illustrative fake code: not from this repo
// BETTER — let the unique index decide, no read-before-write
await prisma.hold.upsert({ where: { slotUnique: slot }, create: slot, update: {} });
```
Real repo: `reserveSlot.handler.ts:67-103` is the BAD shape (check then upsert). The fix uses the existing `SelectedSlots` unique index authoritatively.

---

## 4: N+1 query

```ts
// Illustrative fake code: not from this repo
// BAD — one query per booking to get its attendees
for (const b of bookings) b.attendees = await prisma.attendee.findMany({ where: { bookingId: b.id } });
```
```ts
// Illustrative fake code: not from this repo
// BETTER — one query with a nested select
const bookings = await prisma.booking.findMany({ where, select: { id: true, attendees: true } });
```
Real repo: `createBooking.ts:103-118` fetches related `attendees`/`payment`/`references` in one `select`. Novice instinct: loop and query.

---

## 5: Stale cache key

```ts
// Illustrative fake code: not from this repo
// BAD — key varies with object property order
const key = JSON.stringify({ end, start, eventTypeId }); // different order → cache miss
```
```ts
// Illustrative fake code: not from this repo
// BETTER — canonical, explicit key
const key = `slots:${eventTypeId}:${start}:${end}`;
```
Real repo: `slots/util.ts:108-109` uses `JSON.stringify(args.input)` — works, but its stability depends on property order (risk register #8). The better shape is an explicit key.

---

## 6: Side effect in an unreliable place

```ts
// Illustrative fake code: not from this repo
// BAD — email send inside the DB transaction; a slow SMTP fails the booking
await prisma.$transaction(async (tx) => {
  const b = await tx.booking.create({ data });
  await sendEmail(b); // if this throws, the booking rolls back
});
```
```ts
// Illustrative fake code: not from this repo
// BETTER — commit first, enqueue the effect
const b = await prisma.$transaction((tx) => tx.booking.create({ data }));
await tasker.create("sendEmail", { bookingId: b.id });
```
Real repo: `createBooking.ts:138-145` keeps the transaction to the DB write; effects go through the tasker afterward. (The even-better version enqueues *inside* the txn — the outbox, see [03/04](../03-architecture-and-patterns/04-side-effects-async-and-reliability.md).)

---

## 7: Swallowed error / misleading message

```ts
// Illustrative fake code: not from this repo
// BAD — hides the real cause behind a wrong message
try { await upsertHolds(); } catch { throw new TRPCError({ code: "NOT_FOUND", message: "Event type not found" }); }
```
```ts
// Illustrative fake code: not from this repo
// BETTER — log the real error, return an accurate one
try { await upsertHolds(); }
catch (e) { logger.error("reserve failed", e); throw new TRPCError({ code: "INTERNAL_SERVER_ERROR", message: "Failed to reserve slot" }); }
```
Real repo: `reserveSlot.handler.ts:104-109` is the BAD shape. See [ticket 2](../06-contribution-practice/01-good-first-tickets.md).

---

## 8: `as any` instead of narrowing

```ts
// Illustrative fake code: not from this repo
// BAD — disables the compiler; crashes at runtime
const body = req.body as any;
book(body.eventTypeId, body.start);
```
```ts
// Illustrative fake code: not from this repo
// BETTER — validate, then it's typed
const body = bookingSchema.parse(req.body);
book(body.eventTypeId, body.start);
```
Real repo: `getBookingData.ts:96` parses with a schema; `AGENTS.md` bans `as any`.

---

## 9: Authorization by name, not implementation

```ts
// Illustrative fake code: not from this repo
// BAD — trusts a permissive helper
if (await permissions.can(user, "delete")) doDelete(); // what if can() always returns true?
```
```ts
// Illustrative fake code: not from this repo
// BETTER — enforce the actual role
const member = team.members.find(m => m.userId === user.id);
if (member?.role !== "OWNER" && member?.role !== "ADMIN") throw forbidden();
```
Real repo: this is *literally* the `eventTypes/util.ts:15-20` stub problem. The BETTER shape is the fix.

---

## 10: Boolean-blind time math

```ts
// Illustrative fake code: not from this repo
// BAD — treats adjacent as overlapping; hides free slots
if (busy.end >= slotStart && busy.start <= slotEnd) return true; // off-by-boundary
```
```ts
// Illustrative fake code: not from this repo
// BETTER — half-open intervals; adjacency is NOT a conflict
if (busy.end <= slotStart) continue;   // fully before
if (busy.start >= slotEnd) break;      // fully after (sorted)
return true;
```
Real repo: `checkForConflicts.ts:39-49` is the BETTER shape. The BAD version double-books adjacent meetings.

---

## Drill

For each pair: (a) name the novice instinct in one phrase, (b) find the real repo line that does it right (or, for #3/#5/#7/#9, the line that does it *wrong* and needs the fix), (c) write the one-sentence review comment you'd leave. Take your three best into [04-review-katas.md](04-review-katas.md).
