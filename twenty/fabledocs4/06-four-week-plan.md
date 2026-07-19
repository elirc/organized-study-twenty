# 06 — The four-week apprenticeship plan

A day-by-day thread through all four fabledocs packs. Assumes ~4 focused
hours/day. Adjust pace, not order — the order encodes dependencies.
Every day ends with a lab-notebook entry (three bullets minimum: learned,
surprised, still-confused).

## Week 1 — Orient (read + run)

**Goal:** the system runs on your machine and you can narrate its shape.

- **Day 1** — Setup: `bash packages/twenty-utils/setup-dev-env.sh`,
  `yarn start`, click around a seeded workspace. Read `fabledocs2/00` + `01`
  (architecture). Skim `fabledocs/upskill/01-codebase-cartography/`.
- **Day 2** — Read `fabledocs2/02` (API) + `03` (data access). Trace one real
  request in a debugger: a company list query from resolver to SQL.
- **Day 3** — `fabledocs4/04-sql-tenancy-lab.md`, Parts 1–3. This is the day
  the tenancy model becomes real.
- **Day 4** — SQL lab Parts 4–5. Read ADRs 002–005 (`fabledocs3/`) and notice
  you now recognize everything they cite.
- **Day 5** — Read `fabledocs2/04–06` (testing, errors, observability) +
  ADRs 007–009. Start the ★ flashcards (`fabledocs4/05`). Answer the
  three exit questions of every fabledocs2 section *in writing*.

**Exit criteria:** app runs locally; you can draw the request path
(API → generated schema → query runner → twenty-orm → workspace schema) from
memory; SQL lab exit criteria met.

## Week 2 — Operate (drills + first code)

**Goal:** you can debug this system and pass its quality gates.

- **Day 1** — Fire drills 1 & 2 (`fabledocs4/01`), timed, checkpoints in
  writing before reveals.
- **Day 2** — Fire drills 3–5. Distill your five generalized rules; compare
  with the doc's list.
- **Day 3** — Frontend apprenticeship (`fabledocs4/03`): mental model + the
  Jotai trace exercise.
- **Day 4** — Frontend guided slice (the field-freshness indicator), through
  all quality gates (`lint:diff-with-main`, `typecheck`, tests, Storybook).
- **Day 5** — Testing day: run one integration suite
  (`test:integration:with-db-reset`); read `sync.driver.ts` vs
  `bullmq.driver.ts` line by line; write one new integration test for
  anything you touched this week. ★ flashcards should be ~90% cold now.

**Exit criteria:** five drill rules internalized; one frontend change passing
all gates; you can state without notes what the sync driver hides.

## Week 3 — Build (capstone core)

**Goal:** the outbox exists and survives crashes.

- **Day 1–2** — Capstone Milestone 0 (`fabledocs4/02`): the design doc.
  Two full days is correct; do not compress. Attack it with ADR-010's
  follow-up question ("relay crashes after POST, before mark — what happens?").
- **Day 3–4** — Milestone 1: outbox entity + instance command + writes inside
  the mutation transaction + relay skeleton with `FOR UPDATE SKIP LOCKED`.
- **Day 5** — Milestone 1 acceptance tests under the BullMQ driver (rollback
  leaves no row; worker-kill loses nothing). If red, this bleeds into the
  weekend backlog — the test is the milestone.

**Exit criteria:** M1 acceptance green under real BullMQ; design doc reviewed
(by a person if you have one, adversarially by yourself if not).

## Week 4 — Harden (capstone semantics + synthesis)

**Goal:** at-least-once with dedupability, and you can *say* all of it.

- **Day 1–2** — Milestone 2: surfaced failures, backoff, per-attempt audit
  rows; the "chunk-mates delivered exactly once" regression test.
- **Day 3** — Milestone 3: idempotency keys + dead-letter + failure counter.
  (Milestone 4 is stretch/backlog.)
- **Day 4** — Synthesis: full flashcard deck; rewrite the ADR-010 interview
  answer (`fabledocs3/99`) in your own words ending with "and here's how I
  fixed it" grounded in *your* implementation. Do the same personalization for
  the two ADRs nearest your target role.
- **Day 5** — Mock loop: 90-second spoken answers for five ADRs of your
  choice, recorded and reviewed; one system-design session from
  `fabledocs/upskill/08-interview-prep/04-system-design-from-this-repo.md`;
  write the retro — what in this plan you'd change for the next person.

**Exit criteria:** capstone at mid-complete or better per the rubric; full
deck ≥90%; five ADR answers you can deliver without reading.

## After week 4

You're past what documents can give you. Next moves, in order of value:
pick a ticket-sized real change from
`fabledocs/upskill/06-contribution-practice/`; get your capstone branch
reviewed like a real PR (write the PR description as if for upstream);
and start answering *other people's* questions about this codebase — teaching
is the last rung of the ladder these four packs built.
