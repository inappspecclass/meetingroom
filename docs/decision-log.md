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

## ADR-002 — Supabase Edge Functions own all writes

**Status:** **TEAM-OVERRODE** · 2026-07-28
**Context:** "Backend Node.js, all in Supabase" contains a tension. Supabase Edge
Functions run Deno, not Node. So "all in Supabase" and "Node backend" cannot both
be fully true. The agent surfaced the conflict rather than picking one silently.

**Options considered**

| Option | Trade |
|---|---|
| Separate Node + Fastify service | **Agent proposed this.** Real Node as literally asked. One more deploy target, one more CI pipeline, one more secret store. |
| **Supabase Edge Functions (Deno)** | **Team chose this.** Everything inside Supabase. TypeScript, but Deno rather than Node. |
| React → Postgres RPC directly under RLS | Rejected by both. No server-side place to own the error contract, and the service-role boundary disappears. |

**Agent proposed:** the Node service, on the grounds that the runtime was named
explicitly and that a race test wants a real HTTP surface.

**Team overrode:** Edge Functions.

**Why the team was right:** the HTTP-surface argument does not survive scrutiny —
**an Edge Function is a URL.** The race suite fires N simultaneous `POST`s at it
exactly as it would at Fastify, so the agent's main technical objection was
misplaced. What remains is a straight trade of one runtime against one platform,
and one platform is worth more: a single deploy, a single secret store, no second
CI target, for a one-week scope. The word "Node" in the original request was a
default, not a requirement.

**What it actually cost, stated plainly**

1. **Deno, not Node.** No Node built-ins, no native addons. Nothing in this scope
   needs them. `npm:` specifiers cover the rest, and a `deno.json` import map lets
   the shared zod schema read identically in Vite and Deno.
2. **Cold starts on the write path.** Irrelevant for six rooms; no performance
   claim is made either way.
3. **The real one — no client-side transaction.** See ADR-010. This was not
   obvious when the choice was made, and it is the reason the design changed shape
   rather than just changing runtime.

**Reversal cost:** low. The API contract and the entire transactional core are
runtime-independent; only the HTTP wrapper would move.

**Process note:** this is the shape a Decision Log entry is supposed to have —
agent proposed, team overrode, and the agent's stated reason turned out to be
wrong on inspection. Recording that is the point.

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

**Status:** **TEAM-CONFIRMED** · 2026-07-28
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

## ADR-009 — A series books what it can and reports the rest

**Status:** **TEAM-CONFIRMED** · 2026-07-28
**Context:** A2 #11. A 12-week weekly series where week 7 is already taken.
Settled now, before recurrence is built, because reversing it later costs a
`MODIFIED` delta plus a data migration.

**Options considered**

| Option | Verdict |
|---|---|
| Book the remaining instances, report the skipped ones | **Chosen** |
| All-or-nothing — refuse the whole series | Rejected — a user booking a quarter of standups gets nothing because one week collides |
| A per-request flag | Rejected — doubles the recurrence test matrix and pushes the decision onto a non-technical operator |

**Decision:** partial success with an explicit report of every skipped instance.

**Why:** refusing 11 good bookings over one collision is worse for the user, and
an explicit skip report keeps the outcome honest rather than silent. The failure
mode to avoid is not partial success — it is partial success the user cannot see.

**Consequence for the spec:** `add-recurring-bookings` must include a requirement
that the response enumerates skipped instances with a reason, and a scenario for a
series where *every* instance collides (which is partial success with zero
successes — a distinct case worth its own scenario).

---

## ADR-010 — The transactional unit is a plpgsql function, not the Edge Function

**Status:** AGENT-PROPOSED · 2026-07-28 · consequence of ADR-002
**Context:** `booking-lifecycle` requires an audit event to be written in the same
transaction as the state change it describes, so that a committed state change
without its event is impossible. **`supabase-js` exposes no `BEGIN`/`COMMIT`.**

Two `supabase-js` calls — insert the booking, then insert the audit event — can
commit the first and lose the second. The requirement would be silently violated,
and only the reconciliation test would ever notice.

**Options considered**

| Option | Verdict |
|---|---|
| `book_room()` / `cancel_booking()` as plpgsql functions, called by RPC | **Chosen** — a function body is one implicit transaction |
| Direct Postgres connection from Deno (`deno-postgres`) | Rejected — works, but abandons the Supabase client, connection pooling and RLS story for one feature |
| Two client calls plus a compensating delete on failure | Rejected — a compensating write is not atomicity, and the audit log is append-only, so there is nothing to compensate *with* |
| Relax the requirement | Rejected — the requirement is the point |

**Decision:** plpgsql functions own atomicity. The Edge Function validates and
translates errors.

**Why it matters beyond this project:** the runtime choice looked like a deploy-
target decision and turned out to constrain how an invariant could be enforced.
The spec caught it, because the spec had already committed in writing to
same-transaction audit writes. Had that requirement been implicit, this would have
shipped as a rare, unreproducible missing-audit-event bug.

**Two traps recorded so nobody re-introduces them**

1. **No `EXCEPTION` block in `book_room`.** A plpgsql `EXCEPTION` block rolls back
   the subtransaction, so catching `23P01` there would discard the audit insert
   too — and swallow the one error the system exists to report. Let it propagate.
2. **`security definer` is a privilege boundary.** `search_path` is pinned and
   `execute` is granted to `service_role` only. Granting it to `authenticated`
   would let the browser call it directly and bypass every validation rule.

---

## Pending human decisions

None. ADR-002, ADR-006 and ADR-009 were settled on 2026-07-28.

ADR-010 is a technical consequence of ADR-002 rather than a fresh question, so it
carries the agent's reasoning and is open to challenge at Phase 3 review along
with everything else.

