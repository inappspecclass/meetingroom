# Prompt Journal · Iteration History

Graded artifact #5. The prompt → output → fix loop, logged **live**, including
failures. An empty failure column is a red flag, not a merit.

Entry format:

```
### <n>. <what was asked>
- **Date:** 
- **Prompt (gist):** 
- **Output:** 
- **Wrong because:** 
- **Fix:** 
- **Kept:** 
```

Leave `Wrong because` empty only when the output genuinely needed no correction.
Do not manufacture failures — but do not hide them either.

---

### 1. Draft the Prompt Architect instruction file

- **Date:** 2026-07-27
- **Prompt (gist):** a role definition for a "Prompt Architect" persona, to become
  `CLAUDE.md`.
- **Output:** the file as asked.
- **Wrong because:** nothing in the output, but the *request* conflicted with the
  repo: `AGENTS.md` line 4 instructed that `CLAUDE.md` be a symlink to `AGENTS.md`.
  Writing a real file silently contradicted a standing instruction.
- **Fix:** wrote the real file, and flagged the contradiction rather than
  resolving it silently. `AGENTS.md` was later corrected to drop the symlink
  instruction.
- **Kept:** flag conflicts between a request and the repo's own rules instead of
  quietly picking one. This is the same discipline the spec process encodes.

### 2. Retarget the instructions to the chosen capstone

- **Date:** 2026-07-27
- **Prompt (gist):** we picked meeting-room booking; make the instructions
  project-specific and spec-driven.
- **Output:** project context, invariants, graded artifacts, and a 17-item
  ambiguity backlog derived from the one-paragraph brief.
- **Wrong because:** no correction needed.
- **Kept:** the ambiguity backlog turned out to be the highest-value artifact of
  the session — it is what the proposal is actually built from.

### 3. Write the developer README

- **Date:** 2026-07-27
- **Prompt (gist):** a README explaining spec-driven development and the OpenSpec
  flow.
- **Output:** first draft used `/openspec:proposal` for the propose step, copied
  from `AGENTS.md`.
- **Wrong because:** **that is the legacy command.** The current OpenSpec surface
  is `/opsx:explore → propose → apply → verify → sync → archive`. `AGENTS.md` had
  been wrong since it was written, and the error propagated into the README
  because the README trusted the repo instead of the source.
- **Fix:** checked the OpenSpec CLI and command docs, corrected both files, and
  kept the legacy command noted as still working.
- **Kept:** **an instruction file is not a source of truth about an external
  tool.** Verify tool surfaces against the tool's own docs. This is the same class
  of error the spec process is designed to catch — a confident, plausible,
  unverified claim.

### 4. Document the post-implementation revision loop

- **Date:** 2026-07-27
- **Prompt (gist):** what happens when a requirement or the solution changes after
  implementation?
- **Output:** four cases — in-flight, archived, drift, mechanism-swap.
- **Wrong because:** the first pass treated all four as one procedure, which
  obscured the fact that only the archived case needs a new change with deltas.
- **Fix:** split into four cases with distinct paths, and verified the delta
  semantics against the OpenSpec docs — which surfaced the sharpest trap in the
  format: **`MODIFIED` replaces the entire requirement block, so any scenario you
  omit is deleted.** That would have been discovered the hard way.
- **Kept:** when a question has several shapes, enumerate the shapes before
  writing the procedure. One procedure covering four cases fits none of them.

### 5. Produce the platform proposal, design and tasks

- **Date:** 2026-07-28
- **Prompt (gist):** create the initial spec with Supabase as the platform, React
  frontend, Node backend.
- **Output:** `establish-booking-platform` — proposal, design, tasks, three
  capability deltas, and the seeded artifact files.
- **Wrong because (1):** `npx @fission-ai/openspec init` failed with
  `ECOMPROMISED` (npm cache lock). The temptation was to hand-write
  `openspec/config.yaml` to make the structure look complete.
- **Fix (1):** created only the folders and change files, which `AGENTS.md`
  explicitly permits, and left `openspec init` as task 0.1 for a human. Inventing
  a config schema would have produced a file that looks authoritative and is
  wrong — the worst kind of artifact.
- **Wrong because (2):** the request said "backend Node.js, all in Supabase."
  Those conflict: Supabase Edge Functions are Deno. An agent that did not notice
  would have silently picked one and written it into the design as if settled.
- **Fix (2):** surfaced the conflict as ADR-002 with three options and a
  recommendation, and marked it as needing a human decision.
- **Wrong because (3):** the bootstrap change touches three capabilities, which
  violates `AGENTS.md` §0.5.
- **Fix (3):** kept the violation, because splitting it would put a known
  double-booking defect on `main` — and recorded it as ADR-006 requiring sign-off
  rather than assuming forgiveness.
- **Kept:** three separate instances of the same lesson. **When the honest answer
  is "this conflicts with something," the deliverable is the surfaced conflict,
  not a confident resolution.** An agent's willingness to invent a plausible
  answer is the failure mode this whole process exists to contain.

### 6. Team overrides the write path to Edge Functions

- **Date:** 2026-07-28
- **Prompt (gist):** three decisions put to the team; the write path came back as
  **Supabase Edge Functions (Deno)**, against the agent's Node recommendation.
- **Output:** design revised — Edge Functions replace Fastify.
- **Wrong because:** the agent's stated reason for preferring Node was that a race
  test "needs a real HTTP surface to hammer." **That argument was simply wrong** —
  an Edge Function is a URL, and the race suite fires N concurrent `POST`s at it
  identically. The recommendation was defended with a technical claim that did not
  survive ten seconds of inspection once challenged.
- **Fix:** recorded the override in ADR-002 including *why the agent's reasoning
  was faulty*, not just what changed. Rewrote `design.md`, retargeted 19 tasks.
- **Second-order discovery:** the override exposed a real problem the agent had not
  considered under either option. **`supabase-js` has no multi-statement
  transaction.** The spec requires an audit event to be written in the same
  transaction as its state change, and two client calls can commit a booking and
  lose the event. The transactional unit had to move into Postgres as plpgsql
  functions (ADR-010).
- **Kept:** two lessons, and the second is the valuable one.
  1. **A recommendation is not a conclusion.** The agent produced a confident
     technical justification for a preference. Being asked to defend it revealed it
     was decoration.
  2. **The spec caught what the runtime change hid.** Because "audit event in the
     same transaction" was already written down as a requirement, changing the
     runtime forced the transaction question into the open. Without that
     requirement in writing, this ships as a rare missing-audit-event bug that
     nobody reproduces. **This is the single clearest instance so far of the spec
     doing work no code review would have done.**
- **Also worth noting:** requirements did not change, so no delta was needed — only
  `design.md` and `tasks.md`. That is Case A / Case D of `AGENTS.md` §2 running for
  real, on day one, rather than as a documented hypothetical.

---

## Running observations

Patterns worth carrying into the Learnings Memo:

1. **The agent's failures were never syntax; they were unverified confidence.**
   The legacy slash command, the invented config schema, the "all in Supabase"
   contradiction, the race-test-needs-HTTP argument — each would have produced
   working-looking output containing a false claim. Four for four.
2. **The spec caught what review would have missed.** Writing GIVEN/WHEN/THEN for
   back-to-back bookings forced the half-open interval decision into the open. In
   a code-first build that decision gets made by whichever comparison operator
   someone typed first.
3. **Deferred items must be named.** Every gap here has a row (G-24, recurrence
   evals, rate limiting). Silent truncation reads as coverage.
