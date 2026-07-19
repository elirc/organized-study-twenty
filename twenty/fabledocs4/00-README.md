# fabledocs4 — The Practice Floor

The first three packs teach you to *understand* this codebase. This one makes
you *operate* in it. Everything here is doing, not reading.

## Where this fits

| Pack | Mode | What it gives you |
| --- | --- | --- |
| `fabledocs/upskill/` | Curriculum (53 files) | Cartography, drills, katas, interview prep |
| `fabledocs2/` | Narrative | Principal-engineer onboarding: decisions, warts, incidents |
| `fabledocs3/` | ADRs | The 10 implicit architecture decisions, reverse-engineered |
| `fabledocs4/` | **Practice** | Simulated incidents, a capstone build, hands-on labs, retention |

Prerequisite: fabledocs2 docs 01–03 (or equivalent familiarity). The drills
assume you know what the metadata engine, twenty-orm, and the workspace cache
are; they do not assume you've ever debugged them.

## Contents

1. [01-incident-fire-drills.md](01-incident-fire-drills.md) — five simulated
   on-call incidents. Evidence first, checkpoints before reveals. Do these
   cold, under a timer, like it's real.
2. [02-capstone-reliable-webhooks.md](02-capstone-reliable-webhooks.md) — the
   capstone: design and build transactional-outbox webhook delivery with
   retries and idempotency. This is the real fix for the codebase's most
   documented seam (ADR-010). Milestones, acceptance criteria, test plan.
3. [03-frontend-apprenticeship.md](03-frontend-apprenticeship.md) — the
   server packs left `twenty-front` underserved. Jotai mental model, the
   record store, and a guided feature slice.
4. [04-sql-tenancy-lab.md](04-sql-tenancy-lab.md) — hands-on Postgres against
   the multi-tenant schema layout. You cannot debug this system without
   being comfortable in `psql` across `core`, metadata, and workspace schemas.
5. [05-flashcards.md](05-flashcards.md) — ~50 spaced-repetition cards
   distilled from packs 1–3. Retention is a practice, not a hope.
6. [06-four-week-plan.md](06-four-week-plan.md) — a day-by-day apprenticeship
   schedule threading all four packs, with exit criteria per week.

## Rules of the floor

- **Write before you look.** Every drill has checkpoints. Commit an answer in
  writing before reading the reveal — the gap between your guess and the
  answer is the lesson.
- **Run things.** `bash packages/twenty-utils/setup-dev-env.sh`, then
  `yarn start`. A lab you didn't execute is a blog post you skimmed.
- **Timebox, then ask.** 45 minutes stuck is data; 3 hours stuck is waste.
- **Keep a lab notebook.** One markdown file, append-only, every session:
  what you tried, what surprised you. It becomes your interview story bank.
