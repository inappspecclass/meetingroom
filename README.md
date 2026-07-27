# Meeting-Room Booking — Spec-Driven Build

Developer guide. Read this once before you touch the repo.

**What we are building:** a meeting-room booking system for six rooms in one
office, where two people booking the same slot at the same moment can never both
succeed — plus cancellations, recurring bookings, and a visible audit trail.

**Why it is built this way:** this is InApp Phase 2 capstone option 6. The app is
the vehicle. **The assessed deliverable is the process** — the spec, the
guardrails, the evals, and the learning trail. A polished UI over an untraceable
build fails. A rough build with full traceability passes.

---

## 1. The one rule

> **No code without a spec.**

Every functional change starts as an OpenSpec change proposal, gets human
approval, and only then becomes code. If you find yourself typing
implementation before a proposal is approved, stop — you are off the path.

This is not ceremony. The brief for this project is one ambiguous paragraph, and
the hard part is deciding what "no double-booking" actually means before an
agent confidently guesses for you.

---

## 2. Three files, three audiences

| File | Audience | Role |
|---|---|---|
| `README.md` | **You, a human dev** | How the process works, day to day. Start here. |
| `AGENTS.md` | **Agents + humans** | The operating contract. Binding. Outranks chat. |
| `CLAUDE.md` | **Claude Code** | Project entry point — brief, invariants, graded artifacts. Defers to `AGENTS.md`. |

`AGENTS.md` is a contract, not a tutorial — it tells an agent what it may and may
not do. This README explains the same workflow to a person, with commands.

**The policy surface is protected.** `AGENTS.md`, `CLAUDE.md`, `.claude/`,
`.opencode/`, `openspec/`, `.semgrep/`, `.github/workflows/`, and hook scripts
are never edited as part of a feature task. Change them in a dedicated PR.

---

## 3. Setup

```bash
npm install -g @fission-ai/openspec     # or: npx @fission-ai/openspec
openspec init                           # creates openspec/ and wires up your AI tool
```

`openspec init` creates:

```
openspec/
├── specs/          # living specifications — the source of truth
├── changes/        # in-flight change proposals
└── config.yaml     # project configuration
```

Then pick your agent — Claude Code, opencode, Codex, whatever the team uses.
**The workflow is tool-agnostic; the discipline is not.** Both `AGENTS.md` and
`CLAUDE.md` are read automatically by the common tools.

Useful CLI commands:

| Command | What it does |
|---|---|
| `openspec list` | List changes and specs |
| `openspec show <item>` | Show a change or spec |
| `openspec status` | Artifact completion status for a change |
| `openspec validate <item>` | Structural validation |
| `openspec view` | Interactive dashboard |
| `openspec archive <change>` | Merge deltas into `specs/` and archive the change |

---

## 4. The flow, end to end

```
PROPOSE → LINT SPEC → HUMAN REVIEW → IMPLEMENT → VERIFY → GATED MERGE → ARCHIVE
```

`AGENTS.md` §1 is the authoritative version. Here it is with the commands. It is
a loop, not a line — when a requirement moves you re-enter it rather than patch
around it. See §8.

### Phase 1 — Propose

```
/opsx:explore   # optional: think through the problem first, produces no artifacts
/opsx:propose   # produces proposal.md, specs/, design.md, tasks.md
```

Before you propose, **work the ambiguity backlog in `AGENTS.md` Appendix A2.**
Seventeen open questions hide inside the one-paragraph brief: are intervals
half-open, is there a turnaround buffer, what does the loser of a race receive,
does cancelling a series affect past instances, are rejected attempts audited.
Each one gets answered in the spec or explicitly deferred with a reason in the
Decision Log.

Use the project's capability slugs, exactly these, no synonyms:

`room-inventory` · `booking-lifecycle` · `conflict-resolution` ·
`recurring-bookings` · `audit-trail` · `notifications`

One change = one capability delta. If your change spans two, split it.

### Phase 2 — Lint the spec yourself

Before you ask a human to read it:

- Every SHALL is testable — an engineer could write a failing test for it.
- No weasel words: *appropriate, properly, robust, handle, as needed, etc.*
- Every scenario has concrete GIVEN state, one WHEN trigger, an observable THEN.
- Delta markers (ADDED / MODIFIED / REMOVED) are present and correct.
- One name for one thing across every file.

Prose gate for `proposal.md` and `design.md`: short common words (*use*, not
*utilize*), sentences under 20 words in procedures, active voice, no filler.

Run `openspec validate <change-id>` for the structural check.

### Phase 3 — Human review (hard stop)

Present the change folder and **wait for explicit approval.** Exploration is
fine. File writes into `src/` are not. Silence is not approval.

### Phase 4 — Implement

```
/opsx:apply     # works through tasks.md
```

Work `tasks.md` top to bottom, checking items off. After every file write the
post-write hook runs format, lint, targeted tests and Semgrep. Fix failures
before the next task — do not accumulate a debt pile.

As you go, **log the loop in `docs/prompt-journal.md` live**: the prompt, the
output, what was wrong, what fixed it. Including the failures. Especially the
failures.

### Phase 5 — Verify

```
/opsx:verify    # checks the implementation against the artifacts
```

Every `#### Scenario:` maps to at least one automated test. Name tests after
scenarios — `test_<capability>__<scenario_slug>` — and put the scenario text in
the docstring. Finish `tasks.md` with a verification map:

```
## Verification map
| Requirement | Scenario | Test |
|---|---|---|
```

A change with an unmapped scenario is **not complete**, and you say so out loud.

### Phase 6 — Gated merge

CI is authoritative. Pre-commit runs Semgrep fast rules, a secrets scan,
formatting and slop-lint. The PR gate runs full SAST, dependency scan,
architecture tests, coverage, and every scenario-mapped test. A drift check
fails any PR that touches a spec'd capability in `src/` without a matching
delta.

Fix findings. Never suppress them. No `// nosemgrep`, `# noqa`, or
`@SuppressWarnings` without written human approval in the PR.

### Phase 7 — Archive

```
/opsx:sync      # merge delta specs into openspec/specs/
/opsx:archive   # or: openspec archive <change-id>
```

Specs stay the living source of truth. Delete nothing from git history.

---

## 5. What a change actually looks like

```
openspec/changes/add-conflict-resolution/
├── proposal.md                              # why + what, in plain language
├── design.md                                # decisions, and alternatives rejected
├── tasks.md                                 # numbered, checkable, ends with the verification map
└── specs/conflict-resolution/spec.md        # the delta
```

The delta is the part people get wrong. It is requirements and scenarios, not
prose:

```markdown
## ADDED Requirements

### Requirement: Atomic slot allocation
The system SHALL guarantee that at most one active booking exists for a given
room and overlapping time interval, enforced at the storage layer.

#### Scenario: Two identical bookings submitted simultaneously
- **GIVEN** room `R1` has no active booking on 2026-08-03 from 10:00 to 11:00 IST
- **WHEN** two clients submit a booking for `R1` 10:00–11:00 IST inside the same
  10 ms window
- **THEN** exactly one request returns `201 Created`
- **AND** the other returns `409 Conflict` with code `SLOT_TAKEN`
- **AND** the store holds exactly one active booking for that interval

#### Scenario: Back-to-back bookings do not conflict
- **GIVEN** room `R1` is booked 10:00–11:00 IST
- **WHEN** a client books `R1` 11:00–12:00 IST
- **THEN** the booking succeeds
```

Note what that buys you: scenario 2 settles the half-open-interval question in
writing, so nobody argues about it in code review. That is the whole point.

---

## 6. The eight graded artifacts

Keep these current **as you work**. Reconstructing them the night before is
visible in the diff timestamps and scores badly.

| # | Artifact | Path | The trap |
|---|---|---|---|
| 1 | OpenSpec change proposal | `openspec/changes/<id>/` | No mid-week revision reads as a spec written after the code |
| 2 | Decision Log | `docs/decision-log.md` | Recording the choice but not the *why* |
| 3 | Guardrail Catalogue | `docs/guardrails.md` | Writing a feature list instead of `risk → guardrail → how tested` |
| 4 | Eval Sheet + results | `docs/evals.md` | An acceptance criterion with no eval |
| 5 | Prompt Journal | `docs/prompt-journal.md` | An empty failure column — a red flag, not a merit |
| 6 | Test & Metrics Report | `docs/test-report.md` | Numbers with no summary table |
| 7 | Learnings Memo | `docs/learnings.md` | Generic reflection instead of "here is where the agent silently assumed" |
| 8 | Before vs After Workflow | `docs/workflow-shift.md` | Not revisiting the Block A as-is map |

**Scoring:** process traceability 40% · guardrails and evals 30% · working
end-to-end slice 20% · articulation 10%.

Do not optimise for novelty, UI polish, or feature count. The scorecard cannot
see them. Three features with full traceability beat ten without it.

---

## 7. Non-negotiables for this domain

The six invariants live in `CLAUDE.md` §5. The short version:

- No two active bookings for the same room overlap. Intervals are half-open
  `[start, end)`.
- Concurrent conflicting requests produce exactly one winner and deterministic
  rejections — never two winners, never a crash.
- The audit log is append-only. Every state change has an actor and a timestamp.
- Recurrence expansion is deterministic.
- A cancelled booking frees the slot and stays in the audit trail.

**The store enforces the no-overlap rule, not the service layer and not the UI.**
A read that checks for conflicts followed by an insert is exactly the bug this
project exists to prevent. Use a database constraint, an exclusion constraint,
or a locked compare-and-set — whichever `design.md` names.

Five test classes are mandatory (`AGENTS.md` Appendix A3): **race,
invariant/property, boundary, recurrence, audit-completeness.**

And the one that matters most: **never tune a test to make it pass.** No added
sleeps, no reduced thread count, no retry-until-green, no mocking away the
concurrency. A flaky race test means the system is broken. Escalate it.

---

## 8. When requirements or the solution change

They will. **Specs changing is expected, not a failure** — artifact #1 is graded
partly on showing at least one mid-week spec revision. What fails is a spec and
a codebase that quietly disagree.

The rule underneath all four cases below: **change the spec first, then the
code. Never the other way round, and never both silently.**

### Case A — the change is still in flight (not yet archived)

You are mid-implementation and discover the requirement was wrong, incomplete,
or the customer moved.

1. **Stop implementing.** Do not "just finish this bit first."
2. Run `/opsx:update` to revise the artifacts, keeping `proposal.md`,
   `specs/`, `design.md` and `tasks.md` coherent with each other.
3. Re-run the Phase 2 self-lint and `openspec validate <change-id>`.
4. **If the approved scope moved, go back to Phase 3 and get approval again.**
   Tightening a scenario is not a scope change. Adding a capability is.
5. Record it in `docs/decision-log.md`: what changed, what triggered it, what
   it cost.
6. Update `docs/evals.md` — a changed acceptance criterion means a changed eval.
7. Resume `/opsx:apply`.

**Commit the revision as its own commit.** Do not amend or force-push over the
commit that held the old spec. The revision trail is the deliverable; erasing it
destroys the thing being graded.

### Case B — the change is already archived

The requirement now lives in `openspec/specs/<capability>/spec.md`.

**Never hand-edit `openspec/specs/`.** Those files are the merged output of
archived changes. Editing them directly breaks the audit chain, and the next
`/opsx:sync` may overwrite you.

Instead, raise a **new change** with a delta against the existing spec, and run
the full loop from Phase 1. Four delta operations are available:

```markdown
## ADDED Requirements
### Requirement: <new thing>
...

## MODIFIED Requirements
### Requirement: Atomic slot allocation
<the COMPLETE revised requirement — header, body, and every scenario>

## REMOVED Requirements
### Requirement: <name>
Reason: <why it is going away>

## RENAMED Requirements
- FROM: `### Requirement: Slot locking`
- TO: `### Requirement: Atomic slot allocation`
```

Two rules people get wrong:

- **`MODIFIED` is a replacement, not a diff.** The entire requirement block gets
  swapped for what you write, so include every scenario you still want — the
  ones you leave out are deleted.
- **Renaming is `RENAMED`, not remove-plus-add.** And any `MODIFIED` in the same
  change must reference the *new* name.

If the change alters stored data or live behaviour — a stricter overlap rule
applied to bookings that already exist, say — the proposal must state what
happens to existing records. Migration is part of the change, not a follow-up.

### Case C — the code and the spec disagree

Found in review, or found later by the CI drift check.

**Flag it. Do not silently pick one.** The spec is the source of truth, but
"source of truth" means it has to be *made* true, deliberately:

| What is actually wrong | What you do |
|---|---|
| The code is wrong | Fix the code under the existing spec. No delta needed. |
| The spec is wrong and the built behaviour is correct | Raise a `MODIFIED` delta that documents reality, with the reason in the Decision Log. |
| Nobody is sure which is right | Escalate to a human. This is a requirements question, not a coding one. |

What you never do is leave them disagreeing, or edit whichever one is more
convenient to reach.

### Case D — the requirement holds, but the solution changes

You swap an in-process lock for a database exclusion constraint. Same behaviour,
different mechanism.

- If a requirement names the mechanism — and here it does, because the spec says
  the no-overlap rule is enforced **at the storage layer** — then this *is* a
  spec change. Use Case B.
- If no requirement names it, the requirement stays put but `design.md` must
  still record the new decision and the alternative rejected. Design decisions
  in an archived change folder are frozen, so a post-archive mechanism swap
  needs a new change to carry the new `design.md`.
- Either way, **re-run the race and invariant evals and record the new pass
  rates.** A mechanism swap invalidates the evidence, not just the code.

### Checklist for any post-implementation change

- [ ] Spec updated before code — in-flight via `/opsx:update`, archived via a new change
- [ ] `openspec/specs/` never hand-edited
- [ ] `MODIFIED` blocks contain the complete requirement, every scenario included
- [ ] Re-approved by a human if the scope moved
- [ ] `docs/decision-log.md` — what changed and why
- [ ] `docs/evals.md` — criteria and evals updated, pass rates re-recorded
- [ ] `docs/guardrails.md` — updated if the risk set changed
- [ ] `docs/prompt-journal.md` — still being logged live
- [ ] Verification map regenerated; no scenario left unmapped
- [ ] Revision committed as its own commit, history not rewritten

---

## 9. Your first day

1. Read this file, then `CLAUDE.md`, then `AGENTS.md`.
2. Install and run `openspec init` if the folder is not there yet.
3. Run `openspec list` to see what changes are in flight.
4. Pick a task from an approved change's `tasks.md`. If nothing is approved,
   the work is proposal work, not code work.
5. Before your first prompt to an agent, open `docs/prompt-journal.md` and keep
   it open. Log as you go.

---

## 10. Stop and ask a human when

- A gate fails twice on the same finding after honest fixes.
- The spec is ambiguous, or two requirements contradict each other.
- The task needs a change to the protected policy surface.
- You need a new dependency, a license change, or new data handling.
- You are about to exceed the approved proposal's scope.
- A race or invariant test is flaky.

**When in doubt: propose, don't do.**

---

## References

- [OpenSpec](https://github.com/Fission-AI/OpenSpec) — the spec-driven workflow tool
- [OpenSpec CLI reference](https://github.com/Fission-AI/OpenSpec/blob/main/docs/cli.md)
- [OpenSpec slash commands](https://github.com/Fission-AI/OpenSpec/blob/main/docs/commands.md)
