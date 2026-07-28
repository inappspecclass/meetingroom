# Test & Metrics Report

Graded artifact #6. What ran, what was measured, one summary table.

**Status:** no tests exist yet. `establish-booking-platform` is awaiting Phase 3
approval; implementation starts at task 1.1. This file is the shape the report
takes, not a claim that anything has run.

---

## Summary table

| Class | Suite | Tests | Passed | Failed | Notes |
|---|---|---|---|---|---|
| Race | `tests/race` | — | — | — | Through the `bookings` Edge Function. N ∈ {2, 5, 20}, 50 iterations each |
| Race (function bypassed) | `tests/race` | — | — | — | Direct raw Postgres inserts |
| Race (`book_room` direct) | `tests/race` | — | — | — | Concurrent RPC calls — proves the plpgsql wrapper added no check-then-act window |
| Invariant | `tests/property` | — | — | — | ≥500 generated sequences |
| Boundary | `tests/boundary` | — | — | — | Table-driven, exact error codes |
| Audit | `tests/audit` | — | — | — | Exact event counts, plus forced mid-function failure |
| Unit | `supabase/functions` (`deno test`), `packages/shared` (Vitest) | — | — | — | Two runtimes, two runners |
| **Total** | | **—** | **—** | **—** | |

## Metrics

| Metric | Value | Target |
|---|---|---|
| Line coverage, `supabase/functions` | — | ≥80% |
| Line coverage, `packages/shared` | — | ≥90% |
| Scenarios in delta specs | 39 | — |
| Scenarios with a mapped test | — | 39 |
| Race iterations run | — | 150 |
| Property sequences generated | — | ≥500 |
| Semgrep findings, unresolved | — | 0 |
| Suppressions added | — | 0 |

## What was measured, and what was not

State this honestly. A missing measurement named is worth more than a number
implied.

- **Measured:** correctness under concurrency, invariant preservation under random
  sequences, error-code determinism, audit completeness and immutability.
- **Not measured:** latency, throughput, connection-pool behaviour under load.
  Out of scope — this project optimises for correctness, and no performance claim
  is made.
- **Not measured:** recurrence and DST correctness. Deferred with the capability
  (`add-recurring-bookings`), tracked as G-24.

## Failure log

Every failure that occurred during development, and what fixed it. Do not prune
this — a report with no failures reads as a report that was not written while
working.

| Date | Test | Failure | Root cause | Fix |
|---|---|---|---|---|
| — | — | — | — | — |
