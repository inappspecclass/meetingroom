# room-inventory — delta

## ADDED Requirements

### Requirement: Room registry

The system SHALL maintain a registry of meeting rooms. Each room SHALL have a
unique name, a capacity greater than zero, and an active flag.

The office SHALL be seeded with six rooms.

#### Scenario: The six office rooms are present after seeding

- **GIVEN** a freshly migrated and seeded database
- **WHEN** the room registry is queried
- **THEN** exactly six rooms exist
- **AND** every room has a distinct name and a capacity of at least one

#### Scenario: A duplicate room name is refused

- **GIVEN** a room named `Ganges` already exists
- **WHEN** a second room named `Ganges` is inserted
- **THEN** the insert fails with a unique-constraint violation

#### Scenario: A non-positive capacity is refused

- **GIVEN** the room registry
- **WHEN** a room with capacity `0` is inserted
- **THEN** the insert fails with a check-constraint violation

---

### Requirement: Room listing

The system SHALL expose the room list to any authenticated user, ordered by name,
including each room's capacity and active flag.

#### Scenario: An authenticated user lists rooms

- **GIVEN** six seeded rooms and an authenticated user
- **WHEN** the user requests the room list
- **THEN** all six rooms are returned in ascending name order
- **AND** each entry includes `id`, `name`, `capacity` and `is_active`

#### Scenario: An unauthenticated caller cannot list rooms

- **GIVEN** an unauthenticated caller
- **WHEN** the caller requests the room list
- **THEN** no room data is returned

---

### Requirement: Inactive rooms are not bookable

A room whose active flag is false SHALL NOT accept new bookings. Existing
bookings for that room SHALL be unaffected.

#### Scenario: Booking an inactive room is refused

- **GIVEN** room `R6` exists with `is_active = false`
- **WHEN** a user requests a booking for `R6` on 2026-08-03 from 10:00 to 11:00 IST
- **THEN** the response is `404` with code `ROOM_UNAVAILABLE`

#### Scenario: Deactivating a room leaves its bookings intact

- **GIVEN** room `R5` has an active booking on 2026-08-03 from 10:00 to 11:00 IST
- **WHEN** room `R5` is marked inactive
- **THEN** that booking still exists with status `active`
