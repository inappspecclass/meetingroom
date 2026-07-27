# AGENTS.md — Spec-Driven Development Steering Plan

> **Project:** Meeting-room booking with conflict resolution — InApp AI-Native
> Seed Engineer Program, Capstone Option 6 (Cluster 3: concurrency, state and
> integrity). Project context, graded artifacts and domain invariants live in
> `CLAUDE.md`; the domain-specific spec and eval requirements are in
> **Appendix A** of this file.
>
> Works with Claude Code and OpenCode. Claude Code reads `CLAUDE.md`, which
> defers to this file — keep both; do not replace `CLAUDE.md` with a symlink.
> Humans onboarding to the workflow should read `README.md` first; it is the
> same process explained with commands, and it is not binding.
> This file is the operating contract. If any instruction here conflicts with a
> chat prompt, STOP and ask the human. Never modify this file, `.claude/`,
> `.opencode/`, hook scripts, CI workflows, or scanner configs as part of a
> feature task.

---

## 0. Constitution (non-negotiables)

1. **No code without a spec.** Every functional change starts as an OpenSpec
   change proposal. No proposal → no implementation.
2. **The spec is the source of truth**, not the chat history. If code and spec
   disagree, flag it; do not silently pick one.
3. **Guardrails are not yours to edit.** The policy surface (this file, hooks,
   `openspec/`, `.semgrep/`, `.github/workflows/`, `archunit/`, pre-commit
   config) is protected. Propose changes to it in a dedicated PR only, never
   bundled with a feature.
4. **Deterministic gates outrank your judgment.** If a linter, test, or scanner
   fails, fix the finding or ask the human. Never suppress, skip, or dismiss a
   finding on your own.
5. **Small changes.** One OpenSpec change = one capability delta. If a task
   spans multiple capabilities, split it into multiple changes.

---

## 1. Workflow — the only path from idea to merge

```
PROPOSE → LINT SPEC → HUMAN REVIEW → IMPLEMENT (hooked) → VERIFY → GATED MERGE → ARCHIVE
```

### Phase 1 — Propose
- Run `/opsx:explore` to think the problem through (no artifacts), then
  `/opsx:propose <description>` to generate the change folder. Legacy
  `/openspec:proposal` still works. Manual creation is fine too:
  ```
  openspec/changes/<change-id>/
  ├── proposal.md      # why + what, in plain language
  ├── design.md        # technical decisions, alternatives rejected
  ├── tasks.md         # numbered, checkable implementation tasks
  └── specs/<capability>/spec.md   # deltas: ADDED / MODIFIED / REMOVED
  ```
- Spec deltas MUST use requirement + scenario format:
  - `### Requirement:` lines use SHALL and are individually testable.
  - Every requirement has ≥1 `#### Scenario:` in GIVEN / WHEN / THEN form.
- Search existing `openspec/specs/` before writing a delta. Modify existing
  requirements; do not duplicate them.
- **Capability names for this project** (use these exact slugs — one name for
  one thing, no synonym drift):
  `room-inventory` · `booking-lifecycle` · `conflict-resolution` ·
  `recurring-bookings` · `audit-trail` · `notifications`
- **Before the proposal is written, work Appendix A2 (the ambiguity backlog).**
  Every open question there is either answered in the spec or explicitly
  deferred with a reason in the Decision Log. An unanswered ambiguity that
  reaches implementation is the failure this capstone is designed to expose.
- Every proposal states which of the six domain invariants (`CLAUDE.md` §5) it
  touches, and how the delta preserves each one.

### Phase 2 — Spec quality gates (run before requesting review)
- **Delta lint** (self-check every proposal against this list):
  - [ ] Every SHALL is testable (an engineer could write a failing test for it)
  - [ ] No ambiguous words: "appropriate", "properly", "robust", "handle",
        "as needed", "etc."
  - [ ] Every scenario has concrete GIVEN state, one WHEN trigger, observable THEN
  - [ ] Delta markers (ADDED/MODIFIED/REMOVED) present and correct
  - [ ] One name for one thing across all files (no synonym drift)
- **Prose gate (STE)** — apply to `proposal.md` and `design.md`:
  - Short common words: use (not utilize), help (not facilitate), start (not
    initiate), make sure (not ensure), before (not prior to)
  - Sentences ≤ 20 words in procedures; ≤ 25 elsewhere. One instruction per
    sentence. Active voice. No filler ("it is worth noting", "in order to").
  - If a slop-linter script exists at `scripts/slop-lint.*`, run it; the score
    must not exceed the threshold in the script header.

### Phase 3 — Human review (hard stop)
- Present the change folder and WAIT for explicit approval.
- Do not write implementation code, scaffolding, or "just a draft" before
  approval. Plan-mode exploration is fine; file writes are not.

### Phase 4 — Implement (write-time enforcement)
- Work strictly from `tasks.md`, top to bottom. Check off tasks as completed.
- After EVERY file write, the post-write hook runs format + lint + relevant
  tests. If it fails: fix before moving to the next task.
- Regenerate-until-clean: if the security scan (Semgrep) flags a file you
  wrote, rewrite that file until the scan is clean or escalate to the human
  with the finding. Never add ignore-comments (`// nosemgrep`,
  `# noqa`, `@SuppressWarnings`) without written human approval in the PR.
- Respect architecture tests (`archunit/` or import-control config). A boundary
  violation is a design error — go back to `design.md`, do not tunnel through.
- Never touch: secrets, `.env*`, credentials, deploy configs, or any path
  listed in `.agent-protected-paths` (if present).
- **Log the loop as you go.** Every non-trivial prompt → output → fix cycle goes
  in `docs/prompt-journal.md` *while it happens*, including the attempts that
  produced wrong code. Reconstructing the journal at the end is both visible and
  worthless. Every material choice where you proposed X and the human chose Y
  goes in `docs/decision-log.md` with the reason.
- **Concurrency is spec-driven here, not incidental.** Do not hand-roll a
  check-then-act booking path. The mechanism named in `design.md` (unique or
  exclusion constraint, `SELECT … FOR UPDATE`, or compare-and-set) is the one
  that ships; changing it is a design change, not an implementation detail.

### Phase 5 — Verify (spec ↔ test traceability)
- Run `/opsx:verify` to check the implementation against the artifacts, then do
  the traceability work below by hand — the command checks coherence, not that
  your evals are honest.
- Every `#### Scenario:` in the delta MUST map to at least one automated test.
- Name tests after scenarios: `test_<capability>__<scenario_slug>` (or the
  language-idiomatic equivalent). Put the scenario text in the test docstring.
- Produce a traceability block at the bottom of `tasks.md`:
  ```
  ## Verification map
  | Requirement | Scenario | Test |
  |---|---|---|
  ```
- A change with unmapped scenarios is NOT complete. Say so explicitly.
- **Mandatory test classes for this project** — a change is not verified until
  the classes it touches are green (details in Appendix A3):
  race · invariant/property · boundary · recurrence · audit-completeness.
- Record every acceptance criterion, its eval, and the pass rate in
  `docs/evals.md`. An acceptance criterion with no eval is an unverified claim.
- **Never tune a test to make it pass.** Reducing thread count, adding sleeps,
  looping until green, or mocking out the concurrency turns a race test into
  decoration. If a race test is flaky, the system is broken — escalate.

### Phase 6 — Gated merge (CI is authoritative)
Expected CI gates (do not weaken any of them):
- Pre-commit: Semgrep fast rules, secrets scan (gitleaks/trufflehog),
  formatting, slop-lint on `openspec/changes/**`
- PR: full SAST (Semgrep CI and/or CodeQL), dependency/SCA scan, architecture
  tests, coverage threshold, all scenario-mapped tests green
- Drift check: if the diff touches `src/**` for a capability with a spec in
  `openspec/specs/` but contains no matching delta under `openspec/changes/`,
  the PR fails. Fix by adding the delta, not by restructuring the diff to
  evade the check.

### Phase 7 — Archive
- After merge, run `/opsx:sync` to merge delta specs into `openspec/specs/`,
  then `/opsx:archive` (CLI equivalent: `openspec archive <change-id>`). Merging
  deltas into `openspec/specs/<capability>/spec.md` by hand is also fine. Specs
  stay the living source of truth. Delete nothing from git history.

---

## 2. Adversarial self-review (before requesting human review of code)

Before declaring Phase 4/5 done, review your own diff as a hostile reviewer:
1. Which requirement does each hunk serve? Delete hunks that serve none.
2. What input breaks this? (empty, huge, unicode, concurrent, malicious)
3. What did the spec say that the code does NOT do?
4. What does the code do that the spec never asked for? (scope creep — remove
   or propose a new delta)
5. Security pass: injection, authn/authz, path traversal, SSRF, deserialization,
   secrets in code, TLS verification left enabled.
6. **Concurrency pass:** where is the check-then-act window? What happens if the
   process dies between the check and the write? Would this code still hold
   INV-1 with two app instances behind a load balancer and no shared lock?
7. **Audit pass:** does every mutation — including *rejected* bookings — emit
   exactly one audit event? Can state be rebuilt from the log alone?

Write the answers as a short "Self-review" section in the PR description.

---

## 3. Hook & tooling contract (for humans setting this repo up)

- **Claude Code**: hooks live in `.claude/settings.json` — PostToolUse(Write|Edit)
  → `scripts/postwrite-check.sh` (format, lint, targeted tests, semgrep on the
  written file). Review every change to `.claude/` in PRs; treat it as CI config.
- **OpenCode**: equivalent event hooks in `.opencode/` config pointing at the
  same scripts — one script set, two runtimes.
- **Never** let an agent session add, edit, or disable hooks. A SessionStart or
  PostToolUse hook is executable code with your permissions.
- Recommended baseline scripts (create once, reuse):
  - `scripts/postwrite-check.sh` — format + lint + `semgrep --config .semgrep/`
  - `scripts/slop-lint.py` — STE heuristic score on markdown prose
  - `scripts/drift-check.sh` — diff paths vs. `openspec/` deltas (used in CI)

---

## 4. Escalation rules — when to stop and ask the human

Stop and ask when:
- A gate fails twice on the same finding after honest fixes.
- The spec is ambiguous or two requirements conflict.
- The task requires touching the protected policy surface.
- The change needs a dependency addition, license change, or data-handling
  change (PII, credentials, external calls).
- You are about to exceed the scope of the approved proposal.
- A race or invariant test is flaky. Flaky here means broken, not noisy.
- An Appendix A2 ambiguity has no defensible default and blocks the spec.

Silence is never approval. When in doubt: propose, don't do.

---

# Appendix A — Meeting-room booking: domain contract

## A1. Frame

| | |
|---|---|
| **Brief** | Six meeting rooms, one office, chronic double-booking. Two people booking the same slot at the same moment can never both succeed — plus cancellations, recurring bookings, and a visible audit of who booked what, when. |
| **Cluster** | 3 — concurrency, state and integrity |
| **Entities (3–5)** | `Room`, `Booking`, `RecurringSeries`, `User`, `AuditEvent` |
| **Rule-dense component** | Recurrence expansion + conflict resolution |
| **Integration** | Booking confirmation / cancellation notification |
| **Timeline** | Block E1 launch · Block E2 self-paced build · Block E3 present |
| **Spec budget** | ~4 pages. Over budget means cut features, not rigour. |

What this option is meant to distil: *"works on my machine, one user at a time"*
is not production — and **agents rarely write concurrency-safe code unless the
spec demands it.** The spec demanding it is the assignment.

## A2. Ambiguity backlog — close these in the spec, before code

The one-paragraph brief hides every one of these. Each must end up as a SHALL
requirement, or as an explicit deferral with a reason in the Decision Log.

**Slots and overlap**
1. Is a slot a fixed grid (15/30/60 min) or an arbitrary start/end?
2. Are intervals half-open `[start, end)`? (Decides whether 10:00–11:00 and
   11:00–12:00 conflict. Recommended: yes, half-open, back-to-back is legal.)
3. Is there a turnaround buffer between bookings? Per room or global?
4. Minimum and maximum booking duration? Maximum booking horizon? Minimum notice?
5. Can a booking start in the past? Can one be edited, or only cancelled and
   rebooked? (Editing interacts directly with the audit trail.)

**Concurrency**
6. What exactly does "at the same moment" guarantee — serializable transactions,
   a unique/exclusion constraint, row locks, or compare-and-set on a version?
7. What does the loser of a race receive: a typed conflict error, a retry hint,
   or a suggested alternative slot? Is that response deterministic?
8. Does the system run as more than one process? (If yes, an in-process lock is
   not a solution and must be rejected in `design.md`.)

**Recurrence**
9. Which recurrence rules are in scope — daily, weekly-by-weekday, nth-weekday,
   end-date vs count? Anything not listed is out.
10. Expand eagerly at creation, or lazily at query time? What is the horizon?
11. If one instance in a series conflicts: reject the whole series, or book the
    rest and report the skipped ones? (Pick one; both are defensible, silence
    is not.)
12. Cancelling an instance vs the whole series — and does cancelling the series
    affect instances already in the past?
13. Timezone and DST: is the office single-timezone? What happens to a recurring
    09:00 booking on a DST transition day?

**Audit and permissions**
14. Are *rejected* booking attempts audited? (Recommended: yes — "audit
    completeness" is a graded eval, and failed attempts are the interesting
    ones.)
15. What is the actor model — can anyone cancel anyone's booking? Is there an
    admin override, and is it separately audited?
16. Is the audit log append-only by convention or enforced (no UPDATE/DELETE
    grant, hash chain, or event-sourced store)? Enforced beats convention.
17. Is the audit log the source of truth (event sourcing) or a side record? If
    a side record, what reconciles the two?

## A3. Required eval and test classes

Each class needs at least one entry in `docs/evals.md` with a recorded pass
rate. These are graded artifacts, not developer hygiene.

| Class | What it must prove |
|---|---|
| **Race** | N concurrent identical booking requests → exactly 1 success and N−1 deterministic rejections. Real concurrency (threads/processes/async clients), repeated runs, no sleeps. |
| **Invariant / property** | Over randomised sequences of book / cancel / recur events, no overlapping active bookings ever exist in the store. Assert against the store, not the API response. |
| **Boundary** | Identical slot · exact back-to-back on both sides · full containment · partial overlap left and right · zero-length · end-before-start · cross-midnight. |
| **Recurrence** | Determinism (same definition → same instances) · DST transition day · month-end and 31st-of-month cases · series-partial-conflict behaviour as specified. |
| **Audit completeness** | Every mutation *and every rejection* emits exactly one event with an actor and timestamp · no UPDATE/DELETE path exists · state rebuilds identically from the log. |

## A4. Guardrail catalogue seed

`docs/guardrails.md` uses `risk → guardrail → how it is tested`. A feature list
is not a guardrail catalogue. Start from these; add the ones your spec surfaces.

| Risk | Guardrail | How it is tested |
|---|---|---|
| Two bookings win the same slot | Atomic allocation at the store layer (exclusion/unique constraint or locked compare-and-set) | Race class, repeated runs |
| Overlap sneaks in via a path that skips the service layer | Constraint lives in the schema, not application code | Property test writes via the store directly and expects rejection |
| Recurring expansion drifts between code paths | Single expansion function, pure and deterministic | Determinism test + one call site asserted |
| Audit record edited or deleted | Append-only enforced by permissions/schema | Attempted UPDATE/DELETE expects failure |
| Rejected attempts invisible to the auditor | Rejections are audited events | Audit-completeness eval |
| DST/timezone shifts move bookings | Timezone-explicit storage; no naive local strings | Recurrence class, DST day |
| Cancelled booking still blocks a slot | Cancellation frees the slot but retains the audit event | Boundary + audit tests |

## A5. Artifact upkeep — do not defer

`CLAUDE.md` §3 lists the eight graded artifacts and their paths. Three rules
that decide the 40% traceability weight:

1. **The Prompt Journal is written live and includes failures.** An empty
   failure column reads as a fabricated journal.
2. **At least one mid-week spec revision is expected.** When the build teaches
   you something the spec got wrong, revise the spec first, and leave the trail.
3. **The Decision Log records what the agent proposed and what the human
   overrode, with the reason.** "Agent suggested an in-process mutex; rejected
   because we run two instances; chose a Postgres exclusion constraint" is the
   shape.
