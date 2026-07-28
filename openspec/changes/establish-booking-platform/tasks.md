# Tasks — establish the booking platform

Work top to bottom. Check items off as they complete. The post-write hook runs
format, lint, targeted tests and Semgrep after every file write — fix failures
before starting the next task.

**Do not begin task 1 until this change is approved in Phase 3.**

---

## 0. Bootstrap (human, once)

- [ ] 0.1 Run `openspec init` to generate `openspec/config.yaml` and wire the
      slash commands. The folder structure already exists; `init` should adopt it.
- [ ] 0.2 Install the Supabase CLI and run `supabase start` for a local stack,
      plus `supabase functions serve` so the race suite has a local endpoint.
      Install Deno for `deno test`.
- [ ] 0.3 Create the Supabase project for staging. Record the project ref in the
      Decision Log, not in this file.
- [ ] 0.4 Confirm `.env.example` exists and that `.env*` is git-ignored.

## 1. Schema and constraints

- [ ] 1.1 Migration: `create extension if not exists btree_gist`.
- [ ] 1.2 Migration: `rooms` table with unique name and `capacity > 0` check.
- [ ] 1.3 Seed: the six office rooms in `supabase/seed.sql`.
- [ ] 1.4 Migration: `booking_status` enum and the `bookings` table, including the
      generated `during tstzrange` column using `'[)'`.
- [ ] 1.5 Migration: the `no_overlapping_active_bookings` exclusion constraint,
      partial on `status = 'active'`.
- [ ] 1.6 Migration: interval, minimum-duration and maximum-duration check
      constraints.
- [ ] 1.7 Migration: `audit_events` table.
- [ ] 1.8 Migration: `REVOKE UPDATE, DELETE` on `audit_events`, plus the
      `audit_events_immutable` trigger that raises.
- [ ] 1.9 Migration: enable RLS on all three tables; add read policies only.
- [ ] 1.10 Verify no write policy exists for `authenticated` on `bookings`.

**Gate:** tasks 1.1–1.10 must be green before any API code is written. The
constraint is the product; the API is a wrapper.

## 2. Transactional core (plpgsql)

`supabase-js` has no multi-statement transaction, so atomicity lives in Postgres.
See `design.md` §4.

- [ ] 2.1 Migration: `book_room(p_room_id, p_actor, p_starts_at, p_ends_at)` —
      inserts the booking and its audit event in one function body.
- [ ] 2.2 **No `EXCEPTION` block in `book_room`.** Let `23P01` propagate; catching
      it would roll back the audit insert too.
- [ ] 2.3 Migration: `cancel_booking(p_booking_id, p_actor, p_is_admin)` —
      conditional update plus audit event, one body.
- [ ] 2.4 Migration: `log_rejection(...)` — append-only, called in its own
      transaction after a failure.
- [ ] 2.5 All three `security definer` with `set search_path = public, pg_temp`,
      and `execute` granted to `service_role` only — never `authenticated`.
- [ ] 2.6 Test that `authenticated` cannot call `book_room` directly.

## 3. Shared contracts

- [ ] 3.1 `packages/shared`: zod schema for the booking request, rejecting naive
      timestamps.
- [ ] 3.2 `packages/shared`: the error-code union from `design.md` §5, as a type.
- [ ] 3.3 `deno.json` import map aliasing `zod` → `npm:zod`, so the same source
      file resolves in both Vite and Deno.
- [ ] 3.4 Generate database types from the local Supabase schema.

## 4. Edge Functions (Deno)

- [ ] 4.1 `supabase/functions/_shared`: JWT helper resolving the Supabase token to
      an actor id and admin flag; shared error responder.
- [ ] 4.2 Build check that fails if `SUPABASE_SERVICE_ROLE_KEY` appears in any web
      bundle.
- [ ] 4.3 `bookings` function, `POST` create — validate, then a **single**
      `rpc('book_room')`. No pre-flight overlap query.
- [ ] 4.4 Map SQLSTATE `23P01` to `409 SLOT_TAKEN` including the conflicting
      interval, excluding the other booker's identity.
- [ ] 4.5 On refusal, call `rpc('log_rejection')` in a separate transaction.
- [ ] 4.6 Map every remaining condition in `design.md` §5 to its code.
- [ ] 4.7 `bookings` function, cancel route — `rpc('cancel_booking')`, permission
      check, admin flag.
- [ ] 4.8 Set function secrets for `OFFICE_TIMEZONE` and `BOOKING_HORIZON_DAYS`.
- [ ] 4.9 Confirm no Node built-in or native addon is imported anywhere in
      `supabase/functions/`.

## 5. React web

- [ ] 5.1 Vite + TypeScript scaffold, Supabase auth session.
- [ ] 5.2 Room list from Supabase directly, under RLS.
- [ ] 5.3 Day view per room showing active bookings.
- [ ] 5.4 Booking form posting to the `bookings` Edge Function.
- [ ] 5.5 Render every error code as a distinct message. `SLOT_TAKEN` must read
      as "already booked", not as a generic failure.
- [ ] 5.6 Cancel action, visible only where permitted.

## 6. Tests — the graded part

- [ ] 6.1 Boundary suite for every interval and overlap shape.
- [ ] 6.2 Race suite through the `bookings` function, N = 2, 5, 20, 50 iterations
      each.
- [ ] 6.3 Race suite **bypassing the function**, firing concurrent raw inserts at
      Postgres.
- [ ] 6.4 Race suite firing concurrent `select book_room(...)` calls — proves the
      transactional wrapper did not reintroduce a check-then-act window.
- [ ] 6.5 Property suite: random create/cancel sequences, then assert zero
      overlapping active pairs by querying the store.
- [ ] 6.6 Audit-completeness suite: one event per transition, no null actor on
      success, `UPDATE`/`DELETE` raise, state rebuilds from the log, and a forced
      mid-function failure leaves neither booking nor event.
- [ ] 6.7 Permission suite for cancellation, plus the direct-`rpc` denial test.
- [ ] 6.8 Record every result and pass rate in `docs/evals.md`.
- [ ] 6.9 Write `docs/test-report.md` with the summary table.

## 7. Artifacts before review

- [ ] 7.1 `docs/decision-log.md` current, including anything overridden during
      implementation.
- [ ] 7.2 `docs/guardrails.md` — each entry as risk → guardrail → how tested,
      with the test name filled in.
- [ ] 7.3 `docs/prompt-journal.md` — logged live, failures included.
- [ ] 7.4 Verification map below completed, with real test names.
- [ ] 7.5 Adversarial self-review (`AGENTS.md` §3) written into the PR body.
- [ ] 7.6 Propose `scripts/verify-traceability.sh` in a **separate** PR — it
      compares every `#### Scenario:` title in the delta specs against the
      verification map and fails on a mismatch. Scripts are policy surface
      (`AGENTS.md` §0.3), so it does not ride along with this change.

---

## Verification map

Test names are planned, not yet written. Every scenario in the three delta specs
appears here. **A scenario without a test means this change is not complete.**

**Scenario titles below are copied verbatim from the delta specs.** Keep them
verbatim — that is what lets the traceability check be a script instead of a
careful read. If you rename a scenario, rename it here in the same commit.

### room-inventory

| Requirement | Scenario | Test |
|---|---|---|
| Room registry | The six office rooms are present after seeding | `test_room_inventory__seeded_six_rooms` |
| Room registry | A duplicate room name is refused | `test_room_inventory__duplicate_name_rejected` |
| Room registry | A non-positive capacity is refused | `test_room_inventory__zero_capacity_rejected` |
| Room listing | An authenticated user lists rooms | `test_room_inventory__list_ordered_by_name` |
| Room listing | An unauthenticated caller cannot list rooms | `test_room_inventory__anon_cannot_list` |
| Inactive rooms are not bookable | Booking an inactive room is refused | `test_room_inventory__inactive_room_unavailable` |
| Inactive rooms are not bookable | Deactivating a room leaves its bookings intact | `test_room_inventory__deactivate_preserves_bookings` |

### booking-lifecycle

| Requirement | Scenario | Test |
|---|---|---|
| Booking creation | A valid booking is created | `test_booking_lifecycle__create_valid` |
| Booking creation | An unauthenticated request cannot create a booking | `test_booking_lifecycle__anon_cannot_create` |
| Booking interval validation | End before start is refused | `test_booking_lifecycle__end_before_start` |
| Booking interval validation | A zero-length booking is refused | `test_booking_lifecycle__zero_length` |
| Booking interval validation | A booking shorter than the minimum is refused | `test_booking_lifecycle__below_min_duration` |
| Booking interval validation | A booking of exactly the minimum duration succeeds | `test_booking_lifecycle__exactly_min_duration` |
| Booking interval validation | A booking longer than the maximum is refused | `test_booking_lifecycle__above_max_duration` |
| Booking interval validation | A booking starting in the past is refused | `test_booking_lifecycle__start_in_past` |
| Booking interval validation | A booking beyond the horizon is refused | `test_booking_lifecycle__beyond_horizon` |
| Booking interval validation | A naive local timestamp is refused | `test_booking_lifecycle__naive_timestamp_rejected` |
| Booking cancellation | A user cancels their own booking | `test_booking_lifecycle__cancel_own` |
| Booking cancellation | A different user cannot cancel someone else's booking | `test_booking_lifecycle__cancel_other_forbidden` |
| Booking cancellation | An administrator cancels another user's booking | `test_booking_lifecycle__admin_cancel_flagged` |
| Booking cancellation | Cancelling an already-cancelled booking is refused | `test_booking_lifecycle__cancel_twice_conflict` |
| Booking cancellation | Cancelling a nonexistent booking returns not-found | `test_booking_lifecycle__cancel_missing_not_found` |
| Bookings are not editable | No mutation route exists for booking times | `test_booking_lifecycle__no_edit_route` |
| Every transition is attributed and recorded | A creation writes exactly one audit event | `test_audit__one_event_per_create` |
| Every transition is attributed and recorded | A cancellation writes exactly one audit event | `test_audit__one_event_per_cancel` |
| Every transition is attributed and recorded | A failed state change leaves no audit event behind | `test_audit__rollback_leaves_no_event` |
| Every transition is attributed and recorded | Audit events cannot be altered | `test_audit__update_delete_raise` |
| Every transition is attributed and recorded | Booking state can be rebuilt from the audit log | `test_audit__replay_matches_state` |

### conflict-resolution

| Requirement | Scenario | Test |
|---|---|---|
| Storage-enforced non-overlap | Direct database insert of an overlapping booking is refused | `test_conflict__direct_insert_rejected_23P01` |
| Storage-enforced non-overlap | Identical slot is refused through the API | `test_conflict__identical_slot_409` |
| Storage-enforced non-overlap | Every overlap shape is refused | `test_conflict__all_overlap_shapes_rejected` |
| Half-open interval semantics | Back-to-back bookings both succeed | `test_conflict__back_to_back_allowed` |
| Half-open interval semantics | Booking ending exactly at an existing start succeeds | `test_conflict__abutting_before_allowed` |
| Deterministic resolution of simultaneous requests | Two identical requests submitted simultaneously | `test_conflict__race_two_exactly_one_wins` |
| Deterministic resolution of simultaneous requests | Twenty concurrent requests for the same slot | `test_conflict__race_twenty_exactly_one_wins` |
| Deterministic resolution of simultaneous requests | Concurrent requests for different rooms do not interfere | `test_conflict__race_distinct_rooms_both_win` |
| Cancelled bookings release their slot | Rebooking a cancelled slot succeeds | `test_conflict__rebook_after_cancel` |
| Cancelled bookings release their slot | A cancelled booking does not collide with its replacement | `test_conflict__cancelled_not_counted_in_overlap` |
| Refused bookings are recorded | A slot conflict is audited | `test_conflict__rejection_audited` |

**Cross-cutting:** `test_invariant__no_overlap_under_random_sequences` (property)
covers the `conflict-resolution` non-overlap requirement independently of any
single scenario, and is re-run by every future change.

**Totals:** 39 scenarios, 39 mapped tests, plus 1 property test.
