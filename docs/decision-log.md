# Decision Log

Graded artifact #2. Every material choice: what the agent proposed, what the team
overrode, and **why**. A decision without a reason is not a decision, it is a
coincidence.

**Status key:** `AGENT-PROPOSED` — written by the agent, not yet confirmed by a
human · `TEAM-CONFIRMED` · `TEAM-OVERRODE` · `SUPERSEDED`

Every entry below is `AGENT-PROPOSED` until reviewed in Phase 3. That is
deliberate: an unreviewed agent decision must be visibly distinct from a team
decision, or this artifact is worthless.

---

## ADR-001 — Supabase as the platform

**Status:** AGENT-PROPOSED · 2026-07-28
**Context:** The team chose Supabase, React and Node before the spec existed.
The spec has to establish whether that stack can hold the project's invariants.

**Options considered**

| Option | Verdict |
|---|---|
| Supabase (Postgres + Auth + RLS) | **Chosen** |
| Firebase / Firestore | Rejected — no range type, no exclusion constraint. The no-overlap rule would live in application code or a transaction retry loop. |
| MySQL-backed stack | Rejected — no native range type or exclusion constraint. |

**Decision:** Supabase.

**Why:** Postgres supports `EXCLUDE USING gist` over a `tstzrange`. That makes
INV-1 a storage-engine guarantee rather than an application convention. This is
the single most consequential property of the platform for this project, and it
is the reason the choice survives scrutiny rather than merely being convenient.
Supabase Auth additionally gives attribution a real foreign key.

**Consequence:** we are committed to Postgres-specific DDL. Portability off
Postgres is not a goal; if it ever becomes one, the invariant moves to
application code and the guarantee weakens. Say so out loud at that point.

---

## ADR-002 — Node service owns all writes

**Status:** AGENT-PROPOSED · **needs your decision** · 2026-07-28
**Context:** "Backend Node.js, all in Supabase" contains a tension. Supabase Edge
Functions run Deno, not Node. So "all in Supabase" and "Node backend" cannot both
be fully true.

**Options considered**

| Option | Trade |
|---|---|
| **Separate Node + Fastify service holding the service-role key** | Chosen. Real Node as asked. Gives the race test an HTTP surface. One more deploy target. |
| Supabase Edge Functions (Deno) | Everything inside Supabase, but not Node. Revisit if a single deploy target matters more than the runtime. |
| React → Postgres RPC directly under RLS | Fewest moving parts, but error-contract mapping ends up in plpgsql and the race test has no HTTP surface. |

**Decision:** Node + Fastify service. React reads directly from Supabase under
RLS; all writes go through Node.

**Why:** the runtime was specified explicitly, and the concurrency evidence is
stronger when the race test hits the same path a user does. The invariant does not
depend on this choice — the constraint holds even if Node is bypassed, which is
itself a test (§7 class 2 of `design.md`).

**Reversal cost:** moderate. The API contract survives; the deploy target and auth
plumbing change.

---

## ADR-003 — The no-overlap rule is a database constraint

**Status:** AGENT-PROPOSED · 2026-07-28
**Context:** INV-1 and INV-2. This is the decision the capstone is actually about.

**Options considered**

| Option | Verdict |
|---|---|
| `EXCLUDE USING gist (room_id WITH =, during WITH &&) WHERE (status = 'active')` | **Chosen** |
| Query for overlaps, then insert | Rejected — textbook check-then-act race. Two requests both read "free". |
| In-process mutex | Rejected — we assume more than one API instance. Broken by design, not merely fragile. |
| `SERIALIZABLE` isolation | Rejected — correct but pushes retry-on-serialization-failure into every write path and costs throughput. |
| Advisory lock per room | Rejected — correct but manual. Every future call site must remember it. A constraint is remembered by the database. |

**Decision:** the exclusion constraint, with `btree_gist` for the `room_id WITH =`
component and a partial predicate on `status = 'active'`.

**Why:** it is the only option where forgetting to be careful is not possible. A
new developer adding a second write path in six months inherits the guarantee
without knowing it exists.

**Note on the API:** `POST /bookings` performs **one** insert with no pre-flight
overlap query. A pre-flight check would be dead code implying a guarantee it
cannot give, and would invite future code to trust it.

---

## ADR-004 — Bookings are immutable; change means cancel and rebook

**Status:** AGENT-PROPOSED · 2026-07-28
**Context:** A2 #5. Editable bookings interact badly with the audit trail.

**Decision:** no edit route. A time change is a cancellation plus a new booking.

**Why:** an editable booking makes "who booked this slot, when" ambiguous — the
row that exists is not the row that was created. Cancel-and-rebook keeps every
state transition a discrete, attributable event, which is what INV-4 needs. The
cost is a slightly clumsier UI, which the scorecard does not measure.

**Rejected:** edit-with-history (a revisions table). More machinery than a
one-week scope justifies, and it re-opens the concurrency question on update.

---

## ADR-005 — Timezone handling, and testing DST anyway

**Status:** AGENT-PROPOSED · 2026-07-28
**Context:** A2 #13. The office is in `Asia/Kolkata`, which does not observe DST.

**Decision:** store `timestamptz`, reject naive timestamps at the API boundary,
set `OFFICE_TIMEZONE=Asia/Kolkata`. **And still test recurrence expansion against
`America/New_York`.**

**Why:** the convenient conclusion is "no DST here, so no DST tests." That
conclusion tests the deployment instead of the logic. Expansion code that assumes
fixed 24-hour days is wrong whether or not this office notices, and a future
office in a DST zone would inherit a silent bug. We test the logic.

---

## ADR-006 — The bootstrap change touches three capabilities

**Status:** AGENT-PROPOSED · **needs your sign-off** · 2026-07-28
**Context:** `AGENTS.md` §0.5 requires one change to equal one capability delta.
`establish-booking-platform` touches `room-inventory`, `booking-lifecycle` and
`conflict-resolution`.

**Options considered**

| Option | Verdict |
|---|---|
| Three capabilities in the bootstrap change | **Chosen** |
| `room-inventory` alone first | Rejected — cannot demonstrate any invariant; a table with no behaviour. |
| `booking-lifecycle` before `conflict-resolution` | Rejected — puts a known double-booking defect on `main` between the two merges. |

**Decision:** allow the exception for the bootstrap change only. Every subsequent
change is strictly one capability.

**Why:** the rule exists to keep changes reviewable, not to force a knowingly
broken intermediate state onto the main branch. Splitting here would satisfy the
letter of the rule and damage the thing it protects.

**This is exactly the kind of deviation that must be signed off, not assumed.**

---

## ADR-007 — The audit log is a side record, not the source of truth

**Status:** AGENT-PROPOSED · 2026-07-28
**Context:** A2 #17.

**Decision:** `bookings` holds current state; `audit_events` holds history. A
reconciliation test proves state can be rebuilt from the log.

**Why:** full event sourcing is a larger architectural commitment than a one-week
scope justifies, and it would put the concurrency guarantee behind a projection
rather than a constraint. Keeping state in a constrained table preserves ADR-003.
The replay test gives us most of the auditor-facing benefit without the rebuild.

**Consequence:** the log could in principle diverge from state. That is exactly
why the reconciliation test exists, and why audit writes share a transaction with
their state change.

---

## ADR-008 — Refused bookings are audited

**Status:** AGENT-PROPOSED · 2026-07-28
**Context:** A2 #14.

**Decision:** every refusal writes an audit event with outcome `rejected` and a
reason.

**Why:** "who tried to book this and was turned away" is the question an auditor
actually asks after a double-booking complaint. Logging only successes produces a
log where the interesting event is invisible. The rejection event is written in
its own transaction, because the failed one is aborted.

---

## Pending human decisions

| Ref | Question | Agent recommendation |
|---|---|---|
| ADR-002 | Node service vs Edge Functions vs direct RPC | Node service |
| ADR-006 | Accept the three-capability bootstrap exception? | Accept |
| A2 #11 | Series partial conflict: skip-and-report, or all-or-nothing? | Skip and report |

Nothing in `tasks.md` section 1 or later starts until these are settled.
