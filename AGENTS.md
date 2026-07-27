# AGENTS.md — Spec-Driven Development Steering Plan

> Works with Claude Code and OpenCode. Claude Code also reads `CLAUDE.md` —
> keep a symlink: `ln -s AGENTS.md CLAUDE.md`.
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
- Run `/openspec:proposal <description>` (or create the change folder manually):
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

### Phase 5 — Verify (spec ↔ test traceability)
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
- After merge, run `openspec archive <change-id>` (or merge deltas into
  `openspec/specs/<capability>/spec.md` manually) so specs stay the living
  source of truth. Delete nothing from git history.

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

Silence is never approval. When in doubt: propose, don't do.
