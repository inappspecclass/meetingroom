# Design — establish the booking platform

## 1. Platform decision

| Layer | Choice |
|---|---|
| Database, auth, hosting | **Supabase** (Postgres 15+, Supabase Auth, RLS) |
| Frontend | **React** + TypeScript + Vite, `@supabase/supabase-js` for reads |
| Backend | **Node.js** + TypeScript + Fastify, service-role key, owns all writes |
| Tests | Vitest (unit, integration), `fast-check` (property), Postgres via Supabase CLI local stack |

### Why Supabase, specifically

Not "because it is convenient." Postgres gives us one feature that decides this
project: **an exclusion constraint over a time range.** It turns "no two bookings
overlap" from a rule the application tries to uphold into a rule the storage
engine cannot violate. Every other candidate platform would leave that rule in
application code, which is exactly the failure this capstone exists to expose.

Supabase Auth also gives us a real `auth.users` table, so INV-4 (attribution) has
a foreign key rather than a string.

### Why writes go through Node, not straight from the browser

React talks to Postgres directly for reads under RLS. Writes go through the Node
service, which holds the service-role key.

- The race test needs an HTTP surface to hammer with N simultaneous requests.
  Testing through the real write path is the point; testing the database alone
  would prove the constraint works but not that the API surfaces it correctly.
- Mapping a Postgres error to a stable API contract (`409 SLOT_TAKEN`) is
  business logic, and it belongs in one place.
- The invariant does not depend on this choice. If the Node layer is bypassed
  entirely, the constraint still holds. That is the test in §7, class 2.

## 2. Rejected alternatives

| Alternative | Why rejected |
|---|---|
| **Application-level check-then-act** — query for overlaps, then insert | The exact TOCTOU bug this project exists to prevent. Two requests both read "free" and both insert. |
| **In-process mutex / semaphore** | Holds only inside one process. We assume more than one instance (A2 #8), so this is broken by design, not merely fragile. |
| **`SERIALIZABLE` isolation** | Would work, but converts the problem into retry-on-serialization-failure everywhere and costs throughput. The constraint is cheaper, declarative, and cannot be forgotten at a new call site. |
| **Postgres advisory locks keyed by room** | Correct but manual: every future write path must remember to take the lock. A constraint is remembered by the database. |
| **Supabase Edge Functions (Deno)** | "All in Supabase" is appealing, but Edge Functions are Deno, not Node. You asked for Node. Revisit if we want zero separate deploys. |
| **Writes direct from React via RLS + RPC** | Viable and fewer moving parts, but leaves error-contract mapping in SQL and gives the race test no HTTP surface. Listed as an open question in `proposal.md`. |
| **`audit_events` as event-sourced source of truth** | Rejected for v1 (A2 #17). Full event sourcing is a larger commitment than this scope justifies. We keep the log authoritative *for history* and prove reconciliation by test. |

## 3. Data model

Five entities, matching the sizing rule: `rooms`, `bookings`, `recurring_series`
(table created later by `add-recurring-bookings`), `auth.users`, `audit_events`.

```sql
create extension if not exists btree_gist;   -- required to mix `=` with `&&` in a GiST exclusion constraint

create table rooms (
  id         uuid primary key default gen_random_uuid(),
  name       text not null unique,
  capacity   int  not null check (capacity > 0),
  is_active  boolean not null default true,
  created_at timestamptz not null default now()
);

create type booking_status as enum ('active', 'cancelled');

create table bookings (
  id           uuid primary key default gen_random_uuid(),
  room_id      uuid not null references rooms(id),
  booked_by    uuid not null references auth.users(id),
  starts_at    timestamptz not null,
  ends_at      timestamptz not null,
  status       booking_status not null default 'active',
  created_at   timestamptz not null default now(),
  cancelled_at timestamptz,
  cancelled_by uuid references auth.users(id),

  -- half-open interval: '[)' encodes the A2 #2 decision in the schema itself
  during tstzrange generated always as (tstzrange(starts_at, ends_at, '[)')) stored,

  constraint booking_interval_valid check (ends_at > starts_at),
  constraint booking_min_duration  check (ends_at - starts_at >= interval '15 minutes'),
  constraint booking_max_duration  check (ends_at - starts_at <= interval '8 hours'),

  -- THE invariant. Not application logic.
  constraint no_overlapping_active_bookings
    exclude using gist (room_id with =, during with &&)
    where (status = 'active')
);
```

Three things worth noticing:

1. **`'[)'` is the half-open decision, in the schema.** The ambiguity from A2 #2
   is not resolved in a comment or a code review — it is resolved in the type.
   Back-to-back bookings cannot conflict because the ranges do not overlap.
2. **`where (status = 'active')`** makes the constraint partial, which is what
   lets a cancelled booking free its slot while staying in the table (INV-6).
3. **`during` is generated and stored**, so no code path can write a range that
   disagrees with `starts_at` / `ends_at`.

### Audit log

```sql
create table audit_events (
  id          bigserial primary key,
  occurred_at timestamptz not null default now(),
  actor_id    uuid references auth.users(id),
  action      text not null,      -- booking.created | booking.cancelled | booking.rejected
  outcome     text not null check (outcome in ('success', 'rejected')),
  room_id     uuid,
  booking_id  uuid,
  is_admin_action boolean not null default false,
  reason      text,               -- populated when outcome = 'rejected'
  payload     jsonb not null default '{}'::jsonb
);

-- Append-only, enforced twice: permissions, and a trigger that shouts.
revoke update, delete on audit_events from authenticated, anon;

create function audit_events_immutable() returns trigger language plpgsql as $$
begin
  raise exception 'audit_events is append-only (attempted %)', tg_op;
end $$;

create trigger audit_events_no_mutate
  before update or delete on audit_events
  for each row execute function audit_events_immutable();
```

A trigger that raises, not a rule that silently discards. A silent no-op would
pass a careless test while destroying the guarantee.

Note `actor_id` is nullable only so an unauthenticated rejected attempt can still
be logged. Any `outcome = 'success'` row with a null actor is a bug, asserted in
the audit-completeness eval.

### RLS

```sql
alter table rooms       enable row level security;
alter table bookings    enable row level security;
alter table audit_events enable row level security;

create policy rooms_read    on rooms    for select to authenticated using (true);
create policy bookings_read on bookings for select to authenticated using (true);
```

No `insert`, `update` or `delete` policies for `authenticated`. RLS denies by
default, so the browser cannot write bookings even with a leaked anon key. The
Node service uses the service-role key and bypasses RLS by design.

Audit reads are deliberately closed in this change — `add-audit-trail` opens
them with a considered policy.

## 4. Write path

```
React  ──POST /bookings──▶  Node (Fastify)  ──insert──▶  Postgres
                                  │                          │
                                  │◀──── 23P01 ──────────────┘
                                  ▼
                          409 SLOT_TAKEN + audit event (outcome=rejected)
```

The service does **one** insert. No pre-flight overlap query — a pre-flight check
would be dead code that implies a guarantee it cannot give, and would invite a
future developer to trust it.

Postgres raises SQLSTATE **`23P01`** (`exclusion_violation`) when the constraint
fires. That maps to:

```json
{ "error": { "code": "SLOT_TAKEN", "room_id": "...", "conflicting_interval": "[10:00, 11:00)" } }
```

The conflicting interval is disclosed; the other booker's identity is not
(A2 #7 — privacy).

Cancellation is `update bookings set status = 'cancelled', cancelled_at = now(),
cancelled_by = $actor where id = $id and status = 'active'`. Zero rows affected
means already cancelled or nonexistent, and returns `409` / `404` respectively —
never a silent success.

Both paths write their audit event in the **same transaction** as the state
change, so INV-4 cannot drift. The rejection path writes its audit event in a
separate transaction, because the failed one is aborted.

## 5. Error contract

| Condition | HTTP | Code |
|---|---|---|
| Slot taken | 409 | `SLOT_TAKEN` |
| `ends_at <= starts_at` | 400 | `INVALID_INTERVAL` |
| Duration outside 15 min – 8 h | 400 | `INVALID_DURATION` |
| Start in the past | 400 | `START_IN_PAST` |
| Beyond 90-day horizon | 400 | `BEYOND_HORIZON` |
| Room missing or inactive | 404 | `ROOM_UNAVAILABLE` |
| Cancelling someone else's booking without admin | 403 | `NOT_PERMITTED` |
| Already cancelled | 409 | `ALREADY_CANCELLED` |

Every code is deterministic for a given input. "Deterministic rejection" in INV-2
means this table, not merely "some error."

## 6. Timezone strategy

The office is `Asia/Kolkata`, which does not observe DST. Storage is
`timestamptz` regardless, and the API accepts and returns ISO-8601 with an
explicit offset. Naive local strings are rejected at the boundary.

DST is therefore not a live risk *for this office* — but the recurrence expansion
logic is still wrong if it assumes fixed 24-hour days. So the expansion property
test in `add-recurring-bookings` runs against `America/New_York` as well. We test
the logic, not the current deployment. Recorded as ADR-005.

## 7. Test strategy — mapped to the five mandatory classes

| Class | Implementation |
|---|---|
| **Race** | `POST /bookings` × N (N = 2, 5, 20) fired with `Promise.all` against a real local Supabase. Assert exactly one `201`. Repeat 50 iterations. No sleeps, no retries. |
| **Invariant / property** | `fast-check` generates random sequences of create / cancel operations, applies them, then asserts `select count(*) from bookings a join bookings b on …` finds zero overlapping active pairs. Asserted **against the store**, not API responses. |
| **Boundary** | Identical slot · back-to-back both sides · full containment · partial overlap left and right · zero-length · end-before-start · cross-midnight · exactly 15 min · exactly 8 h · 90-day edge. |
| **Recurrence** | Deferred to `add-recurring-bookings`. |
| **Audit completeness** | Every create, cancel and rejection produces exactly one row · no success row has a null actor · `UPDATE` and `DELETE` both raise · booking state rebuilds from the log alone. |

**A second race test bypasses Node entirely** and fires concurrent inserts
straight at Postgres. If that one fails, the guarantee was never in the database.
That is the test that proves this design rather than merely exercising it.

## 8. Repository layout

```
apps/
├── web/          React + Vite
└── api/          Fastify service
supabase/
├── migrations/   SQL, ordered, forward-only
└── seed.sql      the six rooms
packages/
└── shared/       zod schemas + generated DB types, shared by web and api
tests/
├── race/
├── property/
├── boundary/
└── audit/
```

`packages/shared` exists so the booking request schema has exactly one
definition. Two drifting copies of a validation rule is a silent-divergence bug
waiting to happen.

## 9. Environment

| Variable | Where | Notes |
|---|---|---|
| `SUPABASE_URL` | web, api | |
| `SUPABASE_ANON_KEY` | web | Public by design, safe only because RLS denies writes |
| `SUPABASE_SERVICE_ROLE_KEY` | api only | **Never** in web. Never committed. Never logged |
| `OFFICE_TIMEZONE` | api | `Asia/Kolkata` |
| `BOOKING_HORIZON_DAYS` | api | `90` |

The service-role key bypasses RLS entirely. If it reaches the browser bundle, every
guarantee in this document is void — so it is asserted absent by a build check.
