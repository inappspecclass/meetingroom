# Eval Sheet

Graded artifact #4. Every acceptance criterion is tagged with the eval that
verifies it, and every eval carries a recorded pass rate.

**An acceptance criterion with no eval is an unverified claim.**

Change: `establish-booking-platform` · Status: **not yet run** — this change is
awaiting Phase 3 approval, so every result below reads `—`. Fill these in as
task 5 completes; do not backfill from memory.

---

## Coverage

| AC | Criterion | Eval | Class | Runs | Pass rate |
|---|---|---|---|---|---|
| AC-1 | Six rooms seeded and listable | `test_room_inventory__seeded_six_rooms`, `__list_ordered_by_name` | boundary | — | — |
| AC-2 | Valid booking returns `201` | `test_booking_lifecycle__create_valid` | boundary | — | — |
| AC-3 | N concurrent identical bookings → exactly one `201` | `test_conflict__race_two_exactly_one_wins`, `__race_twenty_exactly_one_wins` | **race** | — | — |
| AC-3 | Guarantee holds with the API bypassed | `test_conflict__direct_insert_rejected_23P01` | **race** | — | — |
| AC-4 | No overlapping active bookings under any sequence | `test_invariant__no_overlap_under_random_sequences` | **invariant** | — | — |
| AC-5 | Back-to-back bookings both succeed | `test_conflict__back_to_back_allowed`, `__abutting_before_allowed` | boundary | — | — |
| AC-6 | Partial, containing and identical overlaps refused | `test_conflict__all_overlap_shapes_rejected`, `__identical_slot_409` | boundary | — | — |
| AC-7 | Cancelled booking frees its slot, stays in the log | `test_conflict__rebook_after_cancel`, `__cancelled_not_counted_in_overlap` | boundary | — | — |
| AC-8 | Only booker or admin cancels; admin flagged | `test_booking_lifecycle__cancel_other_forbidden`, `__admin_cancel_flagged` | boundary | — | — |
| AC-9 | One audit event per transition, with an actor | `test_audit__one_event_per_create`, `__one_event_per_cancel`, `__rollback_leaves_no_event`, `test_conflict__rejection_audited` | audit-completeness | — | — |
| AC-10 | `UPDATE`/`DELETE` on `audit_events` raises | `test_audit__update_delete_raise` | audit-completeness | — | — |
| AC-10 | State rebuilds from the log | `test_audit__replay_matches_state` | audit-completeness | — | — |
| AC-11 | Invalid intervals refused before the database | `test_booking_lifecycle__end_before_start`, `__zero_length`, `__below_min_duration`, `__exactly_min_duration`, `__above_max_duration`, `__start_in_past`, `__beyond_horizon`, `__naive_timestamp_rejected` | boundary | — | — |

**Unmapped criteria: none.** Unmapped scenarios: none — see the verification map
in `openspec/changes/establish-booking-platform/tasks.md`.

---

## Eval protocol — how these are run

Recorded here so a result can be reproduced, and so nobody can quietly change the
conditions to get a better number.

### Race class

- Real local Supabase stack (`supabase start`), not mocks.
- N clients fired with `Promise.all`, no staggering, no sleeps.
- N ∈ {2, 5, 20}. **50 iterations per N.** A pass means 50/50 iterations produced
  exactly one `201`.
- Any `5xx` counts as a failure, not as a rejection.
- **A flaky race test is a failed race test.** It is not retried into a pass; it
  is escalated (`AGENTS.md` §5).

### Invariant class

- `fast-check`, minimum 500 generated sequences per run.
- Operations drawn from: create (random room, random interval within horizon),
  cancel (random existing booking).
- Assertion runs **against the store**, via a self-join for overlapping active
  pairs — not against API responses. An API that lies would otherwise pass.

### Boundary class

- Table-driven. Each row is one interval shape and one expected error code.
- Codes asserted exactly. "Some 4xx" is not a pass — INV-2 requires deterministic
  rejection, which means the specific code.

### Audit-completeness class

- Event counts asserted as exact equality, never "at least one".
- Immutability asserted by expecting a raised error, not by comparing rows after
  the attempt.
- Replay compares reconstructed state to `bookings` for every booking, not a
  sample.

### Recurrence class

Deferred to `add-recurring-bookings`. Listed here so the gap is visible rather
than absent.

---

## Result log

Append one block per run. Never edit a previous block — supersede it.

```
Run: <date> · commit <sha> · change establish-booking-platform
race:        — / —
invariant:   — / —
boundary:    — / —
audit:       — / —
Notes:
```
