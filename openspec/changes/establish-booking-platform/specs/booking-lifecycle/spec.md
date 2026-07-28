# booking-lifecycle — delta

## ADDED Requirements

### Requirement: Booking creation

An authenticated user SHALL be able to book an active room for a time interval,
receiving the created booking on success.

A booking SHALL record the room, the booking user, the start instant, the end
instant and a status of `active`.

#### Scenario: A valid booking is created

- **GIVEN** room `R1` is active and free on 2026-08-03 from 10:00 to 11:00 IST
- **AND** user `alice` is authenticated
- **WHEN** `alice` requests a booking for `R1` from 10:00 to 11:00 IST
- **THEN** the response is `201` with the booking `id` and status `active`
- **AND** the stored booking records `alice` as the booking user

#### Scenario: An unauthenticated request cannot create a booking

- **GIVEN** an unauthenticated caller
- **WHEN** the caller requests a booking for `R1` from 10:00 to 11:00 IST
- **THEN** the response is `401`
- **AND** no booking is created

---

### Requirement: Booking interval validation

The system SHALL refuse a booking whose interval is invalid, with a distinct and
deterministic error code for each violation, before the booking reaches the
non-overlap check.

The rules SHALL be: end strictly after start; duration at least 15 minutes;
duration at most 8 hours; start not in the past; start within 90 days of now.

#### Scenario: End before start is refused

- **GIVEN** room `R1` is active
- **WHEN** a user requests `R1` from 11:00 to 10:00 IST on 2026-08-03
- **THEN** the response is `400` with code `INVALID_INTERVAL`

#### Scenario: A zero-length booking is refused

- **GIVEN** room `R1` is active
- **WHEN** a user requests `R1` from 10:00 to 10:00 IST
- **THEN** the response is `400` with code `INVALID_INTERVAL`

#### Scenario: A booking shorter than the minimum is refused

- **GIVEN** room `R1` is active
- **WHEN** a user requests `R1` from 10:00 to 10:10 IST
- **THEN** the response is `400` with code `INVALID_DURATION`

#### Scenario: A booking of exactly the minimum duration succeeds

- **GIVEN** room `R1` is active and free
- **WHEN** a user requests `R1` from 10:00 to 10:15 IST
- **THEN** the response is `201`

#### Scenario: A booking longer than the maximum is refused

- **GIVEN** room `R1` is active and free
- **WHEN** a user requests `R1` from 09:00 to 18:00 IST
- **THEN** the response is `400` with code `INVALID_DURATION`

#### Scenario: A booking starting in the past is refused

- **GIVEN** the current instant is 2026-08-03 12:00 IST
- **WHEN** a user requests `R1` from 11:00 to 12:00 IST on the same day
- **THEN** the response is `400` with code `START_IN_PAST`

#### Scenario: A booking beyond the horizon is refused

- **GIVEN** the booking horizon is 90 days
- **WHEN** a user requests a booking starting 91 days from now
- **THEN** the response is `400` with code `BEYOND_HORIZON`

#### Scenario: A naive local timestamp is refused

- **GIVEN** room `R1` is active
- **WHEN** a user submits a start value of `2026-08-03T10:00:00` with no offset
- **THEN** the response is `400` with code `INVALID_INTERVAL`

---

### Requirement: Booking cancellation

A booking's creator SHALL be able to cancel it. An administrator SHALL be able to
cancel any booking, and that cancellation SHALL be distinguishable from a
self-cancellation. No other user SHALL be able to cancel a booking.

Cancellation SHALL set status to `cancelled` and record the cancelling actor and
instant.

#### Scenario: A user cancels their own booking

- **GIVEN** `alice` has an active booking `B1`
- **WHEN** `alice` cancels `B1`
- **THEN** the response is `200`
- **AND** `B1` has status `cancelled` with `cancelled_by` set to `alice`

#### Scenario: A different user cannot cancel someone else's booking

- **GIVEN** `alice` has an active booking `B1` and `bob` is not an administrator
- **WHEN** `bob` attempts to cancel `B1`
- **THEN** the response is `403` with code `NOT_PERMITTED`
- **AND** `B1` still has status `active`

#### Scenario: An administrator cancels another user's booking

- **GIVEN** `alice` has an active booking `B1` and `admin` is an administrator
- **WHEN** `admin` cancels `B1`
- **THEN** the response is `200`
- **AND** `B1` has status `cancelled` with `cancelled_by` set to `admin`
- **AND** the audit event for the cancellation is flagged as an administrator action

#### Scenario: Cancelling an already-cancelled booking is refused

- **GIVEN** booking `B1` has status `cancelled`
- **WHEN** its creator cancels `B1` again
- **THEN** the response is `409` with code `ALREADY_CANCELLED`
- **AND** the original cancellation actor and instant are unchanged

#### Scenario: Cancelling a nonexistent booking returns not-found

- **GIVEN** no booking exists with id `00000000-0000-0000-0000-000000000000`
- **WHEN** a user cancels that id
- **THEN** the response is `404`

---

### Requirement: Bookings are not editable

The system SHALL NOT provide a way to change a booking's room, start instant or
end instant after creation. A change SHALL be expressed as a cancellation
followed by a new booking.

#### Scenario: No mutation route exists for booking times

- **GIVEN** the API surface
- **WHEN** a client sends a request to modify the start or end of booking `B1`
- **THEN** the response is `405` or `404`
- **AND** `B1` is unchanged

---

### Requirement: Every transition is attributed and recorded

Every booking creation, cancellation and refusal SHALL produce exactly one audit
event recording the acting user, the instant, the action and the outcome.

An audit event with outcome `success` SHALL always name an actor.

An audit event SHALL be written in the same transaction as the state change it
describes, so a committed state change without its event is impossible.

#### Scenario: A creation writes exactly one audit event

- **GIVEN** an empty audit log
- **WHEN** `alice` successfully books `R1` from 10:00 to 11:00 IST
- **THEN** exactly one audit event exists with action `booking.created`,
  outcome `success` and actor `alice`

#### Scenario: A cancellation writes exactly one audit event

- **GIVEN** `alice` has an active booking `B1` and one audit event
- **WHEN** `alice` cancels `B1`
- **THEN** exactly two audit events exist
- **AND** the second has action `booking.cancelled` and actor `alice`

#### Scenario: A failed state change leaves no audit event behind

- **GIVEN** a booking creation that fails after the audit event is written
  within the same transaction
- **WHEN** the transaction rolls back
- **THEN** no booking and no `booking.created` event exist

#### Scenario: Audit events cannot be altered

- **GIVEN** an audit event exists
- **WHEN** an `UPDATE` or a `DELETE` is attempted on the audit log
- **THEN** the statement raises an error
- **AND** the event is unchanged

#### Scenario: Booking state can be rebuilt from the audit log

- **GIVEN** a sequence of creations and cancellations has been applied
- **WHEN** booking state is reconstructed from the audit log alone
- **THEN** the reconstructed state matches the `bookings` table for every booking
