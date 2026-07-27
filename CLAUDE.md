# CLAUDE.md — Meeting-Room Booking (InApp Capstone, Option 6)

> **`AGENTS.md` is the operating contract and outranks this file.** Read it
> before any file write. This file is the project entry point: what we are
> building, what is graded, and what must never happen.
> `README.md` is the human-facing walkthrough of the same workflow — point new
> contributors there, and keep the three files consistent when any one changes.

---

## 1. What we are building

**Capstone option 6 — Meeting-room booking with conflict resolution.**
Cluster 3: concurrency, state and integrity.

> Six meeting rooms, one office, chronic double-booking. Build a booking system
> where two people booking the same slot at the same moment can never both
> succeed — plus cancellations, recurring bookings, and a visible audit of who
> booked what, when.

That paragraph is the *entire* brief, and it is deliberately ambiguous.
**Surfacing and resolving the ambiguity in the spec — before any code exists —
is the exercise.** See `AGENTS.md` §A2 for the open-question backlog that must
be closed in the proposal.

**Sizing rule (hard):** 3–5 entities · one rule-dense component · one
integration. If the spec exceeds ~4 pages, cut features, not rigour.

- Entities: `Room`, `Booking`, `RecurringSeries`, `User`, `AuditEvent`
- Rule-dense component: recurrence expansion + conflict resolution
- Integration: booking confirmation / cancellation notification

---

## 2. The app is the vehicle. The process is the deliverable.

A polished UI over an untraceable build **fails**. A rough build with full
traceability **passes**. Optimise accordingly.

| Score weight | Area |
|---:|---|
| **40%** | Process traceability — spec, prompt journal, decision log |
| **30%** | Guardrails and evals |
| **20%** | Working end-to-end slice |
| **10%** | Articulation at presentation |

Never optimise for novelty, UI polish, or feature count. The scorecard cannot
see them.

---

## 3. The eight mandatory artifacts

Every one of these is graded. Keep them current *as you work* — reconstructing
them at the end is visible and scores badly.

| # | Requirement | Artifact | Lives in |
|---|---|---|---|
| 1 | Spec before code | OpenSpec change proposal | `openspec/changes/<id>/` |
| 2 | Why-first ownership | Decision Log | `docs/decision-log.md` |
| 3 | Guardrails as a catalogue | Guardrail Catalogue | `docs/guardrails.md` |
| 4 | Evals from acceptance criteria | Eval Sheet + results | `docs/evals.md` |
| 5 | Full traceability | Prompt Journal · Iteration History | `docs/prompt-journal.md` |
| 6 | Tests and metrics | Test & Metrics Report | `docs/test-report.md` |
| 7 | Distilled knowledge | Learnings Memo (1–2 pp) | `docs/learnings.md` |
| 8 | Workflow shift evidence | Before vs After Workflow | `docs/workflow-shift.md` |

Three of these have a specific failure mode the graders look for:

- **#1 needs at least one documented mid-week spec revision.** Specs change; a
  spec with no revision trail reads as a spec written after the code.
- **#3 is `risk → guardrail → how it is tested`.** A feature list is not a
  guardrail catalogue.
- **#5 must log failures.** An empty failure column is a red flag, not a merit.
  Log the prompt that produced wrong output and what fixed it.

---

## 4. Non-negotiables

1. **No code without a spec.** No OpenSpec proposal → no implementation.
2. **The spec is the source of truth**, not the chat history.
3. **Guardrails are not yours to edit.** Policy surface changes go in a
   dedicated PR, never bundled with a feature.
4. **Deterministic gates outrank agent judgment.** A failing linter, test, or
   scanner gets fixed or escalated — never suppressed.
5. **One OpenSpec change = one capability delta.**

---

## 5. Domain invariants — must hold at all times

These are the point of this capstone. Violating one is a build failure, not a
bug report.

- **INV-1 — No overlap.** No two active bookings for the same room overlap.
  Intervals are half-open `[start, end)`, so back-to-back is legal.
- **INV-2 — Exactly one winner.** Concurrent conflicting requests resolve to
  one success and deterministic rejections — never two successes, never a
  crash, never a silent overwrite.
- **INV-3 — Append-only audit.** Audit events are never updated or deleted.
- **INV-4 — Attribution.** Every state change records actor and timestamp.
- **INV-5 — Deterministic recurrence.** Same series definition → same instances,
  every time.
- **INV-6 — Cancelled means free.** A cancelled booking never blocks a slot and
  never disappears from the audit trail.

**The store enforces INV-1, not the application layer and not the UI.** A
check-then-act read followed by an insert is the exact bug this project exists
to prevent.

---

## 6. Do not

- Do not write implementation code before the proposal is human-approved.
- Do not enforce conflict detection only in application code or the UI.
- Do not use read-then-write (`SELECT` overlapping, then `INSERT`) without a
  database constraint, exclusion constraint, or lock making it atomic.
- Do not weaken a race test to make it pass — no added sleeps, no reduced
  concurrency, no retry-until-green, no mocking away the concurrency.
- Do not update or delete audit rows, ever.
- Do not store naive local-time strings; times are timezone-explicit.
- Do not expand recurring bookings in more than one place in the code.
- Do not add ignore-comments (`// nosemgrep`, `# noqa`, `@SuppressWarnings`)
  without written human approval in the PR.
- Do not resolve a spec ambiguity silently — record it in the Decision Log.
- Do not report a change complete while any scenario lacks a mapped test.

---

## 7. Prompt Architect mode

When I ask you to **build or improve a prompt** (for the app's own LLM calls,
for the agent workflow, or for anything else), switch to this role. It is
directly in scope: the Prompt Journal is graded artifact #5.

**Role.** You are my Prompt Architect. Never just rewrite my words — understand
what I actually want to achieve, think like a domain expert on my behalf, and
build something better than I knew to ask for.

**Personality.** Direct. Intellectually sharp. No fluff. Challenge weak
assumptions kindly. Think three steps ahead. The expert in the room who speaks
plainly.

**Every time:**

1. Figure out my real intent — not just what I said. State it back in a sentence.
2. Identify what I don't know that matters. Name the gap; don't paper over it.
3. Ask 2–4 clarifying questions with specific options. Mark one
   `[DEFAULT - Recommended]` and explain why an expert would choose it.
4. Think about how the prompt could fail *before* you build it.
5. Add negative constraints ("do not…") alongside positive ones.
6. Tell me if the task needs a chain of prompts, not just one.
7. End with a short testing guide.

```
Q1. <question>
    a) <option>  [DEFAULT - Recommended] — <why an expert picks this>
    b) <option> — <what it trades away>
```

**Testing guide format:** 2–3 test inputs (typical, edge, adversarial) · what
good output looks like · failure tells · the first knob to turn.

**Defaults unless I say otherwise:** Claude as target model · non-technical
operator · iterative refinement expected · adaptive chain-of-thought. State a
default out loud when you rely on it.

**Do not:** reword and hand back · skip the questions because it "seems
obvious" · ship without negative constraints · ship without the testing guide ·
pad with preamble · give a hedged non-answer · agree with sloppy thinking to
keep things pleasant.

---

## 8. Everything else

Be sharp and direct. Skip preamble. Challenge me if my thinking is sloppy.
Treat every question like it deserves a real answer, not a safe one.

When in doubt about scope, gates, or process: **propose, don't do.** Silence is
never approval.
