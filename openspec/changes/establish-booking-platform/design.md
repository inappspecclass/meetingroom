# Design — establish the booking platform

> **Revised 2026-07-28** — the write path changed from a separate Node service to
> Supabase Edge Functions after the team overrode the agent's recommendation.
> See ADR-002 in `docs/decision-log.md`. No spec delta was required: the
> requirements name a *guarantee* ("enforced at the storage layer"), not a
> runtime, so all 39 scenarios stand unchanged. This is the Case A / Case D path
> in `AGENTS.md` §2 working as intended.

## 1. Platform decision

| Layer | Choice |
|---|---|
| Database, auth, hosting | **Supabase** (Postgres 15+, Supabase Auth, RLS) |
| Frontend | **React** + TypeScript + Vite, `@supabase/supabase-js` for reads |
| Write path | **Supabase Edge Functions** (Deno + TypeScript) |
| Transactional core | **plpgsql functions** invoked by RPC from the Edge Function |
| Tests | Vitest for HTTP/integration/race/property, `deno test` for function-internal units, local Supabase stack |

### Why Supabase, specifically

Not "because it is convenient." Postgres gives us one feature that decides this
project: **an exclusion constraint over a time range.** It turns "no two bookings
overlap" from a rule the application tries to uphold into a rule the storage
engine cannot violate. Every other candidate platform would leave that rule in
application code, which is exactly the failure this capstone exists to expose.

Supabase Auth also gives us a real `auth.users` table, so INV-4 (attribution) has
a foreign key rather than a string.

### Why Edge Functions, and what it costs

The team chose one platform over one runtime. Everything deploys inside Supabase:
no second host, no second CI target, no separate secret store.

Two things that could have been costs, and are not:

- **The race test still has an HTTP surface.** An Edge Function is a URL. The
  concurrency suite fires N simultaneous `POST`s at it exactly as it would at any
  API. This was the main argument for a Node service, and it does not apply.
- **npm packages still work**, via `npm:` specifiers. A `deno.json` import map
  aliases `zod` → `npm:zod` so the shared validation module reads identically in
  Vite and in Deno.

Two things that are genuinely costs, stated rather than glossed:

- **Deno is not Node.** Anything depending on Node built-ins or native addons is
  out. Nothing in this scope needs them.
- **Cold starts sit on the write path.** Irrelevant for six rooms and an office;
  it would matter at scale, and no performance claim is made either way.

### Why writes do not go straight from the browser

React reads Postgres directly under RLS. Writes go through the Edge Function,
which holds the service-role key.

- Mapping a Postgres error to a stable API contract (`409 SLOT_TAKEN`) is business
  logic and belongs in one place.
- RLS grants no write policy at all, so a leaked anon key cannot create a booking.
- The invariant does not depend on this. If the function is bypassed entirely, the
  constraint still holds — which is a test, not a hope (§7, class 2).

## 2. Rejected alternatives

| Alternative | Why rejected |
|---|---|
| **Application-level check-then-act** — query for overlaps, then insert | The exact TOCTOU bug this project exists to prevent. Two requests both read "free" and both insert. |
| **In-process mutex / semaphore** | Holds only inside one isolate. Edge Functions scale to many isolates, so this is broken by design, not merely fragile. |
| **`SERIALIZABLE` isolation** | Would work, but converts the problem into retry-on-serialization-failure everywhere and costs throughput. The constraint is cheaper, declarative, and cannot be forgotten at a new call site. |
| **Postgres advisory locks keyed by room** | Correct but manual: every future write path must remember to take the lock. A constraint is remembered by the database. |
| **Separate Node + Fastify service** | The agent's original recommendation; overridden by the team in favour of a single platform. See ADR-002. |
| **Writes direct from React via RPC under RLS** | No server-side place to own the error contract, and the service-role boundary disappears. |
| **`audit_events` as event-sourced source of truth** | Rejected for v1 (A2 #17). Full event sourcing is a larger commitment than this scope justifies. We keep the log authoritative *for history* and prove reconciliation by test. |
| **Two separate `supabase-js` calls for booking + audit** | **Rejected — this is the important one.** `supabase-js` has no multi-statement transaction, so two calls can commit a booking and lose its audit event. See §4. |

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

`actor_id` is nullable only so an unauthenticated rejected attempt can still be
logged. Any `outcome = 'success'` row with a null actor is a bug, asserted in the
audit-completeness eval.

### RLS

```sql
alter table rooms        enable row level security;
alter table bookings     enable row level security;
alter table audit_events enable row level security;

create policy rooms_read    on rooms    for select to authenticated using (true);
create policy bookings_read on bookings for select to authenticated using (true);
```

No `insert`, `update` or `delete` policies for `authenticated`. RLS denies by
default, so the browser cannot write bookings even with a leaked anon key. The
Edge Function uses the service-role key and bypasses RLS by design.

Audit reads are deliberately closed in this change — `add-audit-trail` opens them
with a considered policy.

## 4. Write path — and why the transaction lives in Postgres

**The constraint that shaped this design:** `supabase-js` exposes no
`BEGIN`/`COMMIT`. Two separate calls — insert the booking, then insert the audit
event — can commit the first and lose the second. That would silently break the
`booking-lifecycle` requirement that a committed state change without its event
is impossible.

So the transactional unit is a **plpgsql function**, whose body is one implicit
transaction. The Edge Function validates and translates; Postgres owns atomicity.

```sql
create function book_room(
  p_room_id uuid, p_actor uuid, p_starts_at timestamptz, p_ends_at timestamptz
) returns bookings language plpgsql security definer as $$
declare v_booking bookings;
begin
  insert into bookings (room_id, booked_by, starts_at, ends_at)
  values (p_room_id, p_actor, p_starts_at, p_ends_at)
  returning * into v_booking;

  insert into audit_events (actor_id, action, outcome, room_id, booking_id, payload)
  values (p_actor, 'booking.created', 'success', p_room_id, v_booking.id,
          jsonb_build_object('starts_at', p_starts_at, 'ends_at', p_ends_at));

  return v_booking;
end $$;
```

No `EXCEPTION` block. If the exclusion constraint fires, the whole function rolls
back and `23P01` propagates to the caller. **Catching it here would be a
mistake** — a plpgsql `EXCEPTION` block rolls back the subtransaction, so the
audit insert would vanish too, and we would have swallowed the one error the
system exists to report.

```
React ──POST /functions/v1/bookings──▶ Edge Function (Deno)
                                             │
                                     rpc('book_room')
                                             │
                                             ▼
                                   Postgres — ONE transaction
                                   booking + audit event
                                             │
                                    ◀── 23P01 on conflict
                                             ▼
                        409 SLOT_TAKEN  +  rpc('log_rejection')  ← separate txn
```

The rejection event is written by a second RPC in its own transaction, precisely
because the first one aborted. `log_rejection` is append-only by construction.

`cancel_booking(p_booking_id, p_actor, p_is_admin)` follows the same shape: a
conditional `update … where id = $1 and status = 'active'`, the audit insert, both
in one function body. Zero rows affected means already cancelled or nonexistent →
`409` / `404`, never a silent success.

Postgres raises SQLSTATE **`23P01`** (`exclusion_violation`), which the Edge
Function maps to:

```json
{ "error": { "code": "SLOT_TAKEN", "room_id": "...", "conflicting_interval": "[10:00, 11:00)" } }
```

The conflicting interval is disclosed; the other booker's identity is not
(A2 #7 — privacy).

**The function performs exactly one `book_room` call. No pre-flight overlap
query.** A pre-flight check would be dead code implying a guarantee it cannot
give, and would invite future code to trust it.

### On `security definer`

`book_room` is `security definer` so it can write while callers hold no write
policy. That makes it a privilege boundary: its `search_path` is pinned
(`set search_path = public, pg_temp`) and `execute` is granted to
`service_role` only, never to `authenticated`. Otherwise the browser could call
it directly and bypass validation.

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
| No or invalid JWT | 401 | `UNAUTHENTICATED` |

Every code is deterministic for a given input. "Deterministic rejection" in INV-2
means this table, not merely "some error."

## 6. Timezone strategy

The office is `Asia/Kolkata`, which does not observe DST. Storage is `timestamptz`
regardless, and the API accepts and returns ISO-8601 with an explicit offset.
Naive local strings are rejected at the boundary.

DST is therefore not a live risk *for this office* — but recurrence expansion is
still wrong if it assumes fixed 24-hour days. So the expansion property test in
`add-recurring-bookings` runs against `America/New_York` as well. We test the
logic, not the current deployment. Recorded as ADR-005.

## 7. Test strategy — mapped to the five mandatory classes

| Class | Implementation |
|---|---|
| **Race** | `POST /functions/v1/bookings` × N (N = 2, 5, 20) fired with `Promise.all` from Vitest against a real local Supabase. Assert exactly one `201`. Repeat 50 iterations. No sleeps, no retries. |
| **Invariant / property** | `fast-check` generates random sequences of create / cancel operations, applies them, then asserts a self-join finds zero overlapping active pairs. Asserted **against the store**, not API responses. |
| **Boundary** | Identical slot · back-to-back both sides · full containment · partial overlap left and right · zero-length · end-before-start · cross-midnight · exactly 15 min · exactly 8 h · 90-day edge. |
| **Recurrence** | Deferred to `add-recurring-bookings`. |
| **Audit completeness** | Every create, cancel and rejection produces exactly one row · no success row has a null actor · `UPDATE` and `DELETE` both raise · booking state rebuilds from the log alone · a forced mid-function failure leaves neither booking nor event. |

**Two tests bypass the Edge Function entirely** and go straight at Postgres:

1. Concurrent raw inserts — if this fails, the guarantee was never in the database.
2. Concurrent `select book_room(...)` calls — proves the transactional wrapper did
   not reintroduce a check-then-act window.

Those are the tests that prove this design rather than merely exercising it.

## 8. Repository layout

```
apps/
└── web/                    React + Vite
supabase/
├── functions/
│   ├── bookings/           POST create, POST :id/cancel
│   ├── rooms/              GET list (thin; React may read directly instead)
│   └── _shared/            zod schemas, error codes, JWT helper
├── migrations/             SQL, ordered, forward-only
└── seed.sql                the six rooms
packages/
└── shared/                 zod schemas + generated DB types
tests/
├── race/
├── property/
├── boundary/
└── audit/
deno.json                   import map: "zod" -> "npm:zod"
```

`packages/shared` holds the booking request schema so there is exactly one
definition. Vite resolves `zod` from `node_modules`; Deno resolves it through the
`deno.json` import map. Same source file, both runtimes. Two drifting copies of a
validation rule is a silent-divergence bug waiting to happen.

## 9. Environment

| Variable | Where | Notes |
|---|---|---|
| `SUPABASE_URL` | web, functions | |
| `SUPABASE_ANON_KEY` | web | Public by design, safe only because RLS grants no write policy |
| `SUPABASE_SERVICE_ROLE_KEY` | functions only | Injected as a function secret. **Never** in web. Never committed. Never logged |
| `OFFICE_TIMEZONE` | functions | `Asia/Kolkata` |
| `BOOKING_HORIZON_DAYS` | functions | `90` |

The service-role key bypasses RLS entirely. If it reaches the browser bundle,
every guarantee in this document is void — so it is asserted absent by a build
check (task 3.2).
