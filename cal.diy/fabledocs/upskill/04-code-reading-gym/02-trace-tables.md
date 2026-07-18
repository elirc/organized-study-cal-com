# Trace Tables

Six data traces. Fill the "value shape" and "risk" columns yourself first (cover the answers), then compare. Columns: step, file/line, value shape, owner, transformation, risk. Anchors as of 2026-07-11.

Covers: one UI→API, one persistence, one auth, one error, one async, one read-path — as required for a complete picture.

---

## Trace 1: Booking form submit → API (UI→API)

| Step | File/line | Value shape | Owner | Transformation | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | `useBookings.ts:136-137` | form state object | client | React Query `mutate` | client data untrusted |
| 2 | `create-booking.ts:5-14` | `BookingCreateBody` | client lib | `post("/api/book/event", data)` | Date→string on wire |
| 3 | `event.ts:44-47` | `req.body` + `creationSource: WEBAPP` | API route | forces source | can't spoof source |
| 4 | `event.ts:50-58` | `{bookingData, bookingMeta}` | API route | wraps for service | `userId: -1` if anon |
| 5 | `RegularBookingService.ts:545-549` | parsed `bookingData` | service | zod parse vs `bookingFields` | reject malformed |
| 6 | `createBooking.ts:138-145` | `Booking` row | write | `$transaction` create | idempotency key applied |

Owner shift: client → API route → service → DB. The trust boundary is step 3-5 — everything before is untrusted.

---

## Trace 2: Booking write & idempotency (persistence)

| Step | File/line | Value shape | Owner | Transformation | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | `createBooking.ts:73-84` | `buildNewBookingData(...)` result | service | assemble create input | shape must match schema |
| 2 | `createBooking.ts:129-136` | credential lookup (if paid) | service | `findFirstOrThrow` | leak layer (repo's job) |
| 3 | `createBooking.ts:138-140` | txn callback | write | begin transaction | atomic scope |
| 4 | `createBooking.ts:141-143` | original booking update (reschedule) | write | `tx.booking.update` | cancels old slot |
| 5 | `createBooking.ts:144` | new `Booking` | write | `tx.booking.create` | — |
| 6 | `booking-idempotency-key.ts:28-35` | `idempotencyKey` string | extension | `uuidv5(start.end.userId)` | overwrites call-site null |
| 7 | `createBooking.ts:145` | `{...booking, userUuid}` | write | return enriched | — |

The invariant: steps 4-5 are one transaction; step 6 runs inside the create via the extension, guaranteeing the key.

---

## Trace 3: Event-type delete authorization (auth)

| Step | File/line | Value shape | Owner | Transformation | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | `_router.ts:104-109` | procedure def | tRPC | `createEventPbacProcedure("eventType.delete",[ADMIN,OWNER])` | looks enforced |
| 2 | `util.ts:120-144` | `event` (id,userId,teamId,users,team.members) | middleware | `findUnique select` | — |
| 3a | `util.ts:151-158` | boolean (personal) | middleware | `userId === ctx.user.id` check | correct IDOR guard |
| 3b | `util.ts:161-167` | `hasPermission` (team) | middleware | `PermissionCheckService.checkPermission` | **stub → always true** |
| 4 | `util.ts:178-196` | boolean | middleware | assigned-users in team check | real guard remains |
| 5 | `delete.handler.ts` | delete result | handler | perform delete | runs even if 3b bypassed |

The lesson lives in 3b: the value is `true` regardless of role. Trace it into `util.ts:15-20` to see why.

---

## Trace 4: A cancellation rejection (error)

| Step | File/line | Value shape | Owner | Transformation | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | `handleCancelBooking.ts:93-105` | parsed input + `bookingToDelete` | handler | zod parse + load | — |
| 2 | `handleCancelBooking.ts:122-127` | `status === CANCELLED`? | handler | guard → `HttpError(400)` | avoids deleting reschedule's event |
| 3 | `handleCancelBooking.ts:155-160` | `endTime < now`? | handler | guard → `HttpError(400)` | can't cancel past |
| 4 | `handleCancelBooking.ts:162-176` | seated non-host? | handler | guard → `HttpError(401)` | authz |
| 5 | `defaultResponder`/error mw | `HttpError` | boundary | → HTTP status + body | one translation point |
| 6 | client | error response | client | React Query `onError` | show message |

Trace the `HttpError` from a service throw to the HTTP status — that's the error boundary in action.

---

## Trace 5: A webhook task from trigger to send (async)

| Step | File/line | Value shape | Owner | Transformation | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | `handleWebhookTrigger.ts:45` | subscriber list | service | `getWebhooks` | — |
| 2 | `scheduleTrigger.ts` | task payload | service | schedule/send decision | — |
| 3 | `tasker.ts:35-41` | `Task` row | tasker | `create(type,payload,{referenceUid})` | dedupe via unique |
| 4 | `task-processor.ts:20-24` | handler fn | processor | resolve + run | at-least-once |
| 5 | `sendPayload.ts:296-299` | signature | send | `HMAC-SHA256(secret, body)` | no-secret fallback |
| 6 | `sendPayload.ts:312-320` | `fetch` response | send | POST, `redirect:"manual"` | SSRF surface |
| 7 | `task-processor.ts:25-34` | succeed/retry | processor | `Task.succeed`/`Task.retry` | poison → churn |

Duplicate deliveries come from step 7 retrying step 6; that's why receivers must be idempotent.

---

## Trace 6: Slot request to cached response (read-path)

| Step | File/line | Value shape | Owner | Transformation | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | `getSchedule.handler.ts:5-7` | input schema | tRPC | resolve service | — |
| 2 | `di/containers/AvailableSlots.ts:42-44` | `AvailableSlotsService` | DI | `container.get` | eager singleton |
| 3 | `util.ts:108-115` | cache key `JSON.stringify(input)` | cache | Redis `get` | key stability |
| 4 | `util.ts:113-120` | cached or null | cache | timeout→log→fallback | graceful degrade |
| 5 | `util.ts` `_getAvailableSlots` | slot list | service | schedule − busy + limits | timezone correctness |
| 6 | `checkForConflicts.ts:39-49` | boolean per slot | service | interval overlap | adjacency edges |
| 7 | `util.ts` set cache | slots | cache | Redis `set` TTL 2000ms | 2s staleness |

The 2-second TTL (step 7) is why two users can briefly see the same "free" slot — advisory, not a lock.

---

## How to use

Do all six with the answer columns covered. For each, the "risk" column is the interview gold — practice saying the risk aloud. Then take Trace 3 and 5 into [04-review-katas.md](04-review-katas.md).
