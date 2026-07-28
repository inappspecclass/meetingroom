# Learnings Memo

Graded artifact #7. One to two pages. Where the agent silently assumed, where the
spec caught it, and what we would spec differently next time.

**Status:** in progress. Written at the end of Block E2, but the raw material is
collected as we go — see the running observations in `docs/prompt-journal.md`. Do
not wait until the last day; by then the interesting details are gone.

---

## 1. Where the agent silently assumed

*Fill from the Prompt Journal. Candidates already recorded:*

- Trusted `AGENTS.md` for an external tool's command surface instead of checking
  the tool's own docs. The claim was plausible, confident and wrong.
- Would have hand-written an `openspec/config.yaml` to make the scaffold look
  complete, inventing a schema it had never seen.
- Was asked for a "Node backend, all in Supabase" — a contradiction, since Edge
  Functions are Deno. Silently picking one was the easy path.
- Defended its Node recommendation with a technical claim ("the race test needs a
  real HTTP surface") that was false — an Edge Function is a URL. The
  justification was decoration on a preference, and it took being overridden to
  expose that.

## 2. Where the spec caught it

*Fill as implementation proceeds. Already visible:*

- Writing a GIVEN/WHEN/THEN scenario for back-to-back bookings forced the
  half-open interval question into the open before any code existed. In a
  code-first build, that decision gets made by whichever comparison operator
  someone typed first, and nobody ever states it.
- Requiring "how it is tested" on every guardrail row exposed which risks we had
  named but could not actually verify.
- **The strongest instance so far.** Changing the runtime from Node to Deno looked
  like a deploy-target decision. Because the spec had already committed in writing
  to "the audit event is written in the same transaction as its state change," the
  change forced a question nobody had asked: `supabase-js` has no transaction. Two
  client calls can commit a booking and lose its event. Without that requirement
  written down, this ships as a rare missing-audit-event bug that nobody
  reproduces. No code review would have caught it, because the code would have
  looked correct.

## 3. What we would spec differently

*To be written.* Prompts:

- Which ambiguity did we resolve too early, and pay for?
- Which did we defer that we should have settled on day one?
- Which requirement turned out untestable as written?
- Where did the spec describe a mechanism when it should have described a
  guarantee — or the reverse?

## 4. On concurrency specifically

The capstone's stated lesson is that agents rarely write concurrency-safe code
unless the spec demands it. Record the evidence either way:

- What did the agent produce when asked for a booking endpoint *without* the
  constraint in the spec? (Worth running deliberately as an experiment.)
- Did it reach for a check-then-act query, an application lock, or a constraint?
- How much of the safety came from the spec, and how much from the platform?

## 5. Process

- Which gate caught the most real problems?
- Which gate was pure friction?
- Would we keep the artifact set as-is for the next project?
