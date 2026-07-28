# conflict-resolution — delta

## ADDED Requirements

### Requirement: Storage-enforced non-overlap

The system SHALL guarantee that no two bookings with status `active` for the same
room occupy overlapping time intervals, and SHALL enforce this guarantee in the
database schema rather than in application code.

Any write path that reaches the `bookings` table — including a direct SQL insert
that bypasses the API entirely — SHALL be subject to this guarantee.

#### Scenario: Direct database insert of an overlapping booking is refused

- **GIVEN** room `R1` has an active booking on 2026-08-03 from 10:00 to 11:00 IST
- **WHEN** an operator inserts a row directly into `bookings` for `R1` from 10:30
  to 11:30 IST with status `active`, bypassing the API
- **THEN** the insert fails with SQLSTATE `23P01`
- **AND** the store still holds exactly one active booking for `R1` that day

#### Scenario: Identical slot is refused through the API

- **GIVEN** room `R1` has an active booking on 2026-08-03 from 10:00 to 11:00 IST
- **WHEN** a second user requests `R1` from 10:00 to 11:00 IST
- **THEN** the response is `409` with code `SLOT_TAKEN`
- **AND** the response names the conflicting interval
- **AND** the response does not disclose the identity of the first booker

#### Scenario: Every overlap shape is refused

- **GIVEN** room `R1` has an active booking on 2026-08-03 from 10:00 to 11:00 IST
- **WHEN** a user requests `R1` for each of 09:30–10:30, 10:30–11:30,
  10:15–10:45 and 09:00–12:00 IST
- **THEN** every one of those four requests returns `409 SLOT_TAKEN`

---

### Requirement: Half-open interval semantics

The system SHALL treat a booking interval as half-open, including its start
instant and excluding its end instant, written `[start, end)`.

Two bookings that share only a boundary instant SHALL NOT be treated as
overlapping.

#### Scenario: Back-to-back bookings both succeed

- **GIVEN** room `R1` has an active booking on 2026-08-03 from 10:00 to 11:00 IST
- **WHEN** a user requests `R1` from 11:00 to 12:00 IST
- **THEN** the booking is created and the response is `201`
- **AND** room `R1` holds two active bookings for that day

#### Scenario: Booking ending exactly at an existing start succeeds

- **GIVEN** room `R1` has an active booking on 2026-08-03 from 10:00 to 11:00 IST
- **WHEN** a user requests `R1` from 09:00 to 10:00 IST
- **THEN** the booking is created and the response is `201`

---

### Requirement: Deterministic resolution of simultaneous requests

When two or more requests that would overlap arrive concurrently, the system
SHALL create exactly one booking and SHALL refuse every other request with a
deterministic error.

The system SHALL NOT rely on process-local locking, since more than one API
instance may run.

#### Scenario: Two identical requests submitted simultaneously

- **GIVEN** room `R1` has no active booking on 2026-08-03 from 10:00 to 11:00 IST
- **WHEN** two clients submit a booking for `R1` from 10:00 to 11:00 IST
  concurrently, with no ordering between them
- **THEN** exactly one request returns `201`
- **AND** the other returns `409` with code `SLOT_TAKEN`
- **AND** the store holds exactly one active booking for that room and interval

#### Scenario: Twenty concurrent requests for the same slot

- **GIVEN** room `R1` has no active booking on 2026-08-03 from 14:00 to 15:00 IST
- **WHEN** 20 clients submit a booking for that room and interval concurrently
- **THEN** exactly one request returns `201`
- **AND** 19 requests return `409 SLOT_TAKEN`
- **AND** no request returns a `5xx` status

#### Scenario: Concurrent requests for different rooms do not interfere

- **GIVEN** rooms `R1` and `R2` are both free on 2026-08-03 from 10:00 to 11:00 IST
- **WHEN** two clients concurrently book `R1` and `R2` for that interval
- **THEN** both requests return `201`

---

### Requirement: Cancelled bookings release their slot

A booking with status `cancelled` SHALL NOT prevent a new booking for the same
room and interval, and SHALL remain present in the store for audit purposes.

#### Scenario: Rebooking a cancelled slot succeeds

- **GIVEN** room `R1` had a booking on 2026-08-03 from 10:00 to 11:00 IST that is
  now `cancelled`
- **WHEN** another user requests `R1` from 10:00 to 11:00 IST
- **THEN** the booking is created and the response is `201`
- **AND** the cancelled booking row still exists with status `cancelled`

#### Scenario: A cancelled booking does not collide with its replacement

- **GIVEN** room `R1` has a cancelled booking and a new active booking for the
  same interval
- **WHEN** the store is queried for overlapping active bookings for `R1`
- **THEN** no overlapping pair is found

---

### Requirement: Refused bookings are recorded

Every refused booking attempt SHALL produce exactly one audit event with outcome
`rejected` and a reason, whether the refusal came from validation or from the
non-overlap guarantee.

#### Scenario: A slot conflict is audited

- **GIVEN** room `R1` has an active booking on 2026-08-03 from 10:00 to 11:00 IST
- **WHEN** a second user's request for the same interval is refused
- **THEN** exactly one audit event exists with action `booking.rejected`,
  outcome `rejected` and reason `SLOT_TAKEN`
- **AND** that event records the requesting actor, the room and the interval
