# Establish the booking platform

**Change ID:** `establish-booking-platform`
**Capabilities touched:** `room-inventory`, `booking-lifecycle`, `conflict-resolution`
**Status:** DRAFT — awaiting human review (Phase 3)

---

## Why

Six meeting rooms, one office, chronic double-booking. Today two people can book
the same room for the same slot and both believe they succeeded. The cost is a
meeting displaced in front of a client, and nobody can say who booked first
because no record survives.

The fix is not a nicer calendar. It is making overlapping bookings
*impossible to store*, and making every booking decision attributable.

## What this change does

This is the foundation change — the walking skeleton. It delivers the smallest
system that proves the hard part works:

1. Six rooms exist and can be listed.
2. A user can create a booking and cancel their own booking.
3. **Two people booking the same slot at the same moment cannot both succeed.**

The third item is the whole point. It is delivered as a database constraint, not
application logic — see `design.md`.

## What this change does not do

Deferred to follow-on changes, listed in the roadmap below:

- Recurring bookings (series creation, expansion, per-instance cancellation)
- The audit query surface — this change writes audit events, but exposes no
  read API and makes no replay guarantee
- Email or calendar notifications
- Room attributes beyond name and capacity; no capacity enforcement
- Turnaround buffers between bookings
- Suggested alternative slots when a booking is refused
- Admin UI

## Scope note — a deliberate exception, needs your sign-off

`AGENTS.md` §0.5 says one change equals one capability delta. This change touches
three. The reason: a foundation change that cannot create a booking cannot
demonstrate the no-overlap invariant, and shipping `booking-lifecycle` before
`conflict-resolution` would put a known double-booking defect on `main`.

Splitting further produces a change that is either untestable or knowingly
broken. Every change after this one is strictly one capability. Recorded as
ADR-006 in `docs/decision-log.md`.

## Resolved ambiguities

The brief is one paragraph. `AGENTS.md` Appendix A2 lists 17 questions hiding
inside it. Resolutions below become SHALL requirements in the spec deltas.

| # | Question | Resolution |
|---|---|---|
| 1 | Fixed slot grid or arbitrary times? | Arbitrary start and end, minute precision |
| 2 | Are intervals half-open? | **Yes — `[start, end)`.** Back-to-back bookings are legal |
| 3 | Turnaround buffer? | None in v1. Deferred, not forgotten |
| 4 | Duration and horizon limits? | Min 15 min, max 8 h, booking horizon 90 days, no minimum notice |
| 5 | Bookings in the past? Editable? | No past starts. **Not editable** — cancel and rebook, which keeps the audit trail honest |
| 6 | What guarantees "same moment"? | Postgres `EXCLUDE USING gist` constraint. Not a lock, not a transaction isolation level |
| 7 | What does the loser receive? | `409` with code `SLOT_TAKEN` and the conflicting interval. Deterministic. The other booker's identity is **not** disclosed |
| 8 | More than one process? | Yes, assumed. In-process locks are therefore rejected outright |
| 9 | Which recurrence rules? | Deferred to `add-recurring-bookings`. Daily and weekly-by-weekday only when it lands |
| 10 | Eager or lazy expansion? | Deferred. Design intent: eager, 90-day horizon |
| 11 | One instance of a series conflicts? | Deferred. **Needs your decision — see open questions** |
| 12 | Cancel instance vs series? | Deferred |
| 13 | Timezone and DST? | Office is `Asia/Kolkata` (no DST). All timestamps stored `timestamptz`. Expansion logic still gets a DST test against a DST-observing zone — see ADR-005 |
| 14 | Are rejected attempts audited? | **Yes.** A refused booking is the interesting event |
| 15 | Actor model? | Booker cancels own. Admin cancels any, separately flagged in the audit event. Nobody else |
| 16 | Append-only enforced or convention? | **Enforced** — `REVOKE UPDATE, DELETE` plus a trigger that raises. Testable |
| 17 | Is the audit log the source of truth? | No. `bookings` holds state, `audit_events` is the log. A reconciliation test proves state rebuilds from the log |

## Open questions for you

Two items where an expert could reasonably go either way, and guessing would be
expensive to reverse later:

1. **Series partial conflict (A2 #11).** A 12-week weekly series where week 7 is
   already taken. Book the other 11 and report the skip, or refuse the whole
   series? *Recommended: book the rest and report the skip* — refusing 11 good
   bookings over 1 collision is worse for the user, and the report keeps it
   honest. Reversing this later is a `MODIFIED` delta plus a data migration.
2. **Write path (ADR-002).** Node service holding the service-role key, or
   Postgres RPC called directly from React under RLS? *Recommended: Node
   service* — you asked for a Node backend, and it gives the race test a real
   HTTP surface to hammer. The invariant is enforced in the database either way.

## Roadmap

| Order | Change | Capability |
|---|---|---|
| 1 | `establish-booking-platform` | *this change* |
| 2 | `add-audit-trail` | `audit-trail` — query surface, completeness, replay |
| 3 | `add-recurring-bookings` | `recurring-bookings` |
| 4 | `add-booking-notifications` | `notifications` |

`add-audit-trail` comes before recurrence deliberately: recurrence multiplies the
number of state transitions, and the audit surface should exist before the volume
arrives.

## Acceptance criteria

| ID | Criterion | Eval class |
|---|---|---|
| AC-1 | Six rooms are seeded and listable | boundary |
| AC-2 | A valid booking is created and returns `201` | boundary |
| AC-3 | N concurrent identical bookings yield exactly one `201` and N−1 `409 SLOT_TAKEN` | **race** |
| AC-4 | No overlapping active bookings exist in the store under any event sequence | **invariant** |
| AC-5 | Back-to-back bookings both succeed | boundary |
| AC-6 | Partial, containing and identical overlaps are all refused | boundary |
| AC-7 | A cancelled booking frees its slot and remains in the audit log | boundary |
| AC-8 | Only the booker or an admin can cancel; admin cancellation is flagged | boundary |
| AC-9 | Every create, cancel and rejection emits exactly one audit event with an actor | audit-completeness |
| AC-10 | `UPDATE` or `DELETE` on `audit_events` raises an error | audit-completeness |
| AC-11 | Invalid intervals are refused before reaching the database | boundary |

Recurrence and DST evals arrive with `add-recurring-bookings`. AC-4's property
test is written now and re-run by every later change.
