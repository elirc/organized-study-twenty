# Twenty CRM — Upskill Curriculum

A training lab built on this repository for one learner: a **junior fullstack JS engineer** (React/Node/TS CRUD experience) who wants to reach mid-level fast, build senior judgment, and pass interviews for mid-level fullstack roles. Every module teaches two things at once: how **this** codebase works (with exact file anchors) and the **transferable pattern** you can recognize and articulate in any other repo — including in an interview room.

## What this repo is

Twenty is an open-source CRM (a Salesforce/HubSpot alternative) built as an Nx monorepo managed with Yarn 4. The backend is NestJS + TypeORM + PostgreSQL + Redis, exposing a **metadata-driven GraphQL API**: the schema is not written by hand but *generated per workspace* from object/field metadata stored in the database ([workspace-schema.factory.ts](../../packages/twenty-server/src/engine/api/graphql/workspace-schema.factory.ts#L23-L68)). The frontend is React 18 + Jotai + Apollo + Linaria, driven by the same metadata so users can define custom objects and fields at runtime. Multi-tenancy is per-workspace, enforced down at the ORM layer where a custom query builder injects permission checks and row-level predicates before any SQL runs ([workspace-select-query-builder.ts](../../packages/twenty-server/src/engine/twenty-orm/repository/workspace-select-query-builder.ts#L342-L352)). Background work (webhooks, workflows, email sync) runs on BullMQ workers. This is a production-grade system with real reliability tradeoffs worth studying — and worth talking about in interviews.

## Learning tracks

| Track | Directory | What you get |
| --- | --- | --- |
| Cartography | [01-codebase-cartography/](01-codebase-cartography/README.md) | Maps, reading order, glossary, 6 traced end-to-end flows |
| Stack mastery | [02-stack-and-language-mastery/](02-stack-and-language-mastery/README.md) | TS/Node/React/Nest mental models anchored to real files |
| Architecture | [03-architecture-and-patterns/](03-architecture-and-patterns/README.md) | Boundaries, persistence, auth, async reliability, 14 pattern cards, critique |
| Reading gym | [04-code-reading-gym/](04-code-reading-gym/README.md) | Annotation drills, trace tables, fake-code contrasts, review katas |
| Quality | [05-quality-engineering/](05-quality-engineering/README.md) | Testing, debugging, performance, security, observability |
| Contribution | [06-contribution-practice/](06-contribution-practice/README.md) | 15 junior tickets, 10 mid tickets, 5 senior projects, katas |
| Career | [07-career-and-collaboration/](07-career-and-collaboration/README.md) | Review mindset, PRs/RFCs, maintainer communication |
| **Interview prep** | [08-interview-prep/](08-interview-prep/README.md) | 41+ question cards (mostly repo-anchored), system design, STAR stories, cram plan |
| Reference | [09-reference/](09-reference/) | Commands, risk register, rubrics, verification log |

## How to use it

- **One weekend:** [00-fast-track.md](00-fast-track.md) end to end.
- **Two weeks:** Fast track → all of 01 → 03/05-key-flows-adjacent files → 2 drills per day from 04 → one junior ticket from 06.
- **Eight weeks:** Weeks 1–2 as above; weeks 3–4: all of 02 and 03; weeks 5–6: 05 plus two mid-level tickets; weeks 7–8: one senior project from 06 plus all of 08.
- **Ongoing contribution:** live in 06 and 07; use 09-reference/risk-register.md as a source of PR ideas.

**Recommended paths by profile**

- **Brand-new junior:** 00 → 01 (all) → 02 → 04-annotation drills → one Easy ticket.
- **Junior who knows React/Node:** 00 → 01/05-key-flows → 03 → 04 → 06 junior tickets.
- **Mid-level, new to the repo:** 01/01-system-map + 01/05-key-flows → 03/06-architecture-critique → 06 mid tickets.
- **Senior doing architecture review:** 03/06-architecture-critique → 09-reference/risk-register.md → 03/04-side-effects-async-and-reliability.md.
- **Candidate with an interview in two weeks:** go directly to [08-interview-prep/07-two-week-cram-plan.md](08-interview-prep/07-two-week-cram-plan.md).

## Conventions in these docs

- **File anchors** look like `[file.ts](../../packages/...#L10-L20)` — path plus line range, verified at writing time (see [09-reference/verification-log.md](09-reference/verification-log.md)). Line numbers drift as the repo changes; the path and symbol name are the durable part.
- **Fake code is labeled.** Any snippet not from this repo starts with `// Illustrative fake code: not from this repo`. Everything else is real.
- **Commands** are marked __verified__ (actually run while writing these docs) or __inferred__ (read from manifests/CI but not executed). Most here are __inferred__ — this clone was studied without installing dependencies.
- **Claims** about behavior come from reading code. Anything not fully confirmed is labeled *investigate* or *possible risk*, never asserted as a bug.
- **Drills** end with self-grading criteria (Basic / Solid / Strong). Grade yourself honestly; the gap between Solid and Strong is exactly the gap between junior and mid-level interview answers.

## The mindset ladder

- **Junior asks:** "How do I make it work?"
- **Mid-level asks:** "Is this the right pattern? What breaks if I do it the obvious way?"
- **Senior asks:** "What does this commit us to? Who pays the cost? How do we reduce the blast radius?"

Interviews for mid-level roles test exactly the second and third questions. When an interviewer asks "how would you add webhooks to your API?", they are not checking whether you know what an HTTP POST is — they are checking whether you mention signing, timeouts, retries, and what happens when the receiver is down. This repo gives you concrete, defensible answers to all of that. Use them.
