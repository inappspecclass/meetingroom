# Guardrail Catalogue

Graded artifact #3. Every entry is **risk → guardrail → how it is tested.**
A feature list is not a guardrail catalogue. If a row has no test, it is a wish.

Scope: `establish-booking-platform`. Rows are added as later changes land.

---

## Concurrency and integrity

| # | Risk | Guardrail | How it is tested |
|---|---|---|---|
| G-01 | Two users win the same slot | `EXCLUDE USING gist (room_id WITH =, during WITH &&) WHERE (status='active')` — allocation is atomic in the storage engine | `test_conflict__race_two_exactly_one_wins`, `test_conflict__race_twenty_exactly_one_wins` — 50 iterations each, real concurrency, no sleeps |
| G-02 | A future write path skips the service layer and inserts an overlap | The constraint is in the schema, so no code path can evade it | `test_conflict__direct_insert_rejected_23P01` — inserts straight into Postgres, expects SQLSTATE `23P01` |
| G-03 | Overlap enters through an unforeseen sequence of operations | Same constraint, verified exhaustively rather than by example | `test_invariant__no_overlap_under_random_sequences` — `fast-check` random create/cancel sequences, asserts zero overlapping active pairs **by querying the store** |
| G-04 | An in-process lock gives false confidence under multiple instances | In-process locking is banned by ADR-003; nothing in the write path holds one | Code review + `test_conflict__direct_insert_rejected_23P01` (passes with the API absent entirely) |
| G-05 | `during` disagrees with `starts_at` / `ends_at` | `during` is a `GENERATED ALWAYS ... STORED` column; it cannot be written independently | `test_conflict__all_overlap_shapes_rejected` plus a schema assertion that the column is generated |
| G-06 | Back-to-back bookings wrongly refused (or overlaps wrongly allowed) by off-by-one interval logic | Half-open `'[)'` encoded in the range constructor itself, not in application comparison code | `test_conflict__back_to_back_allowed`, `test_conflict__abutting_before_allowed` |
| G-07 | A cancelled booking keeps blocking its slot | Constraint is partial on `status = 'active'` | `test_conflict__rebook_after_cancel`, `test_conflict__cancelled_not_counted_in_overlap` |

## Audit and attribution

| # | Risk | Guardrail | How it is tested |
|---|---|---|---|
| G-08 | Audit records edited or deleted to hide a mistake | `REVOKE UPDATE, DELETE` **and** a trigger that raises on `UPDATE`/`DELETE` — loud, not silent | `test_audit__update_delete_raise` |
| G-09 | A silent no-op guardrail passes a careless test while destroying the guarantee | The trigger raises an exception; a `DO INSTEAD NOTHING` rule was explicitly rejected | `test_audit__update_delete_raise` asserts an error is raised, not merely that the row is unchanged |
| G-10 | A state change commits without its audit event | Audit write shares the transaction with the state change | `test_audit__one_event_per_create`, `test_audit__rollback_leaves_no_event` |
| G-11 | Refused attempts are invisible to an auditor | Refusals write an event with outcome `rejected` and a reason, in their own transaction | `test_conflict__rejection_audited` |
| G-12 | An event exists with no actor | `outcome = 'success'` rows must name an actor | `test_audit__one_event_per_create` asserts a non-null actor |
| G-13 | History and state drift apart | Reconciliation: state is rebuilt from the log and compared | `test_audit__replay_matches_state` |
| G-14 | Editing a booking makes "who booked this" ambiguous | No edit route exists; change is cancel plus rebook (ADR-004) | `test_booking_lifecycle__no_edit_route` |

## Authorisation and data exposure

| # | Risk | Guardrail | How it is tested |
|---|---|---|---|
| G-15 | A user cancels someone else's booking | Permission check: creator or admin only; admin action separately flagged | `test_booking_lifecycle__cancel_other_forbidden`, `test_booking_lifecycle__admin_cancel_flagged` |
| G-16 | The browser writes bookings directly using the anon key | RLS enabled with read policies only; no write policy for `authenticated`, and RLS denies by default | `test_room_inventory__anon_cannot_list` plus an RLS assertion that an anon insert into `bookings` fails |
| G-17 | The service-role key leaks into the frontend bundle | Build check greps the web bundle for the key and fails the build | Build-time check in CI (task 3.2) |
| G-18 | A conflict response leaks who holds the slot | The `409` body names the interval, never the other user | `test_conflict__identical_slot_409` asserts no user identifier in the body |
| G-19 | Secrets committed | `.env*` git-ignored; gitleaks/trufflehog in pre-commit | Pre-commit secrets scan |

## Input and time correctness

| # | Risk | Guardrail | How it is tested |
|---|---|---|---|
| G-20 | Nonsense intervals reach the database | Zod validation at the boundary **and** check constraints in the schema — both layers, deliberately | `test_booking_lifecycle__end_before_start`, `__zero_length`, `__below_min_duration`, `__above_max_duration` |
| G-21 | Naive local timestamps stored, losing the offset | API rejects timestamps without an explicit offset; storage is `timestamptz` | `test_booking_lifecycle__naive_timestamp_rejected` |
| G-22 | Unbounded booking horizon lets someone reserve a room for 2031 | 90-day horizon enforced in the API | `test_booking_lifecycle__beyond_horizon` |
| G-23 | Bookings created in the past, corrupting the day view | Start-in-past refused | `test_booking_lifecycle__start_in_past` |
| G-24 | Recurrence expansion assumes fixed 24-hour days | Expansion tested against a DST-observing zone even though the office has none (ADR-005) | Deferred to `add-recurring-bookings` — **tracked, not silently dropped** |

## Process guardrails

| # | Risk | Guardrail | How it is tested |
|---|---|---|---|
| G-25 | A race test is quietly weakened until it passes | No sleeps, no retries, no reduced concurrency, no mocking; flaky means broken and gets escalated | Review of test diffs; iteration count and `Promise.all` shape asserted in the test itself |
| G-26 | Scanner findings suppressed instead of fixed | No `nosemgrep` / `noqa` / `@SuppressWarnings` without written approval in the PR | Pre-commit Semgrep; PR review |
| G-27 | An acceptance criterion ships with no eval | Verification map in `tasks.md` must cover every scenario | Drift check + Phase 5 gate |

---

## Known gaps

Stated rather than hidden. Silent truncation reads as full coverage.

- **G-24** is deferred to `add-recurring-bookings` because recurrence is not in
  this change.
- Room capacity is stored but not enforced against attendee counts. Out of scope
  for v1, no guardrail claimed.
- No rate limiting on `POST /bookings`. A user could book every slot in the
  horizon. Real risk, deliberately out of scope, no guardrail claimed.
