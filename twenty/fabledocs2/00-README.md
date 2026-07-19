# Onboarding: Welcome to Twenty (from the person who has the most scar tissue)

Hey. I'm the principal engineer on this codebase, and this doc pack is what I wish
someone had handed me. It's not the marketing tour — you can get that from the
website. This is the candid version: why things are shaped the way they are, which
decisions I'd defend in front of a review board, and which ones I'd quietly redo if
I had a free quarter.

## How to read this

Six sections, in dependency order. Read 01 first no matter what — everything else
in this system is downstream of the metadata engine, and nothing will make sense
until you internalize that **the CRM's data model does not exist at compile time**.

| Doc | Subsystem |
| --- | --- |
| [01-architecture.md](01-architecture.md) | The metadata engine, the monorepo, and why the server is really two applications |
| [02-api-design.md](02-api-design.md) | Runtime-generated GraphQL, the REST facade, and the Zapier coupling I'm not proud of |
| [03-data-access.md](03-data-access.md) | twenty-orm, per-workspace schemas, dynamic DDL migrations |
| [04-testing-strategy.md](04-testing-strategy.md) | Why we test against a real Postgres and what the sync queue driver hides from you |
| [05-error-handling.md](05-error-handling.md) | Exception taxonomy, filters, and the webhook delivery incident |
| [06-observability.md](06-observability.md) | Logger, metrics, audit events, Sentry — and the fire-and-forget problem |

Each section ends with three questions. Take them seriously — they're the questions
I'll ask you in our 1:1 before I let you review PRs in that area. If you can't
answer them from the code, go back and read the files cited. Every file path in
these docs is real; open them side by side with the prose.

## Ground rules while you ramp

- **Never touch a workspace schema table with raw TypeORM.** Always go through
  `WorkspaceRepository` (`packages/twenty-server/src/engine/twenty-orm/repository/workspace.repository.ts`).
  Section 03 explains why this rule exists and what happened when someone broke it.
- **Run `npx nx typecheck twenty-server` before you believe anything works.**
  Half our safety net is the type system; the other half is integration tests
  against a real database.
- **When you change entity files, generate an instance command**
  (`npx nx run twenty-server:database:migrate:generate --name <name> --type <fast|slow>`).
  Committed migration `up`/`down` logic is append-only. No exceptions.
- **The metadata cache is not your friend until you understand it.** If your
  change "doesn't take effect," it's almost always `workspace-cache`, not your code.

Setup is one script: `bash packages/twenty-utils/setup-dev-env.sh`. It's idempotent.
If it fights you, ping me before you improvise — the failure modes are all known.

— your friendly principal, who wrote most of the bugs described herein
