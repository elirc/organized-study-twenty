# 04 — Testing strategy: why we pay for a real Postgres on every CI run

## What it is and why it exists

The stated pyramid (it's in `CLAUDE.md`): ~70% unit, ~20% integration, ~10% E2E.
The honest version: **the unit tests protect logic, the integration tests protect
the metadata engine, and the metadata engine is where the company-ending bugs
live.** So while unit tests outnumber everything, the integration suite is the
one I actually trust before a release.

The layers:

- **Unit tests** — Jest, colocated `__tests__/` and `*.spec.ts` files throughout
  `twenty-server` and `twenty-front`. Run a single file with
  `npx jest path/to/test.spec.ts --config=packages/PROJECT/jest.config.mjs` —
  always prefer single-file runs while developing; the full suite is a CI thing.
- **Integration tests** — `packages/twenty-server/test/integration/`, organized
  into `graphql/`, `metadata/`, `rest/`, `billing/`, `auth-adjacent suites`, with
  shared `utils/` and codegen'd clients (`test/integration/graphql/codegen/`).
  Run via `npx nx run twenty-server:test:integration:with-db-reset` — note the
  `with-db-reset`; these hit a real Postgres seeded by
  `src/engine/workspace-manager/dev-seeder/`.
- **E2E** — `packages/twenty-e2e-testing/`, Playwright against a running stack.
- **Storybook interaction tests** — `twenty-front`'s component-level safety net
  (`npx nx storybook:test twenty-front`).

## The hard decisions

### 1. Real Postgres in integration tests vs. mocking the data layer

The rejected alternative was mocking twenty-orm (or Postgres via something like
pg-mem). We tried a version of this early. It was worthless — I mean that
precisely: the tests passed and the system broke, which is worse than having no
tests. The metadata engine's whole job is generating SQL DDL and dynamic queries;
a mock that faithfully reproduces Postgres's DDL semantics, constraint behavior,
and error codes *is* Postgres, minus the fidelity.

So integration tests run against the real thing, with a dev-seeded workspace, and
we eat the cost: CI minutes, `with-db-reset` slowness, and occasional
test-ordering flakes when someone forgets tests share a database. The mitigation
for local pain is in `CLAUDE.md`: don't run the whole suite locally; run your file.

Still bothers me: reset granularity. A full DB reset between *suites* (not tests)
means tests within a suite can couple through leftover data. We've been bitten
maybe four times by a test that only passes after another test runs. The fix —
per-test workspace isolation, spinning a fresh workspace schema per test — is
architecturally *easy* here (workspaces are cheap by design!) and we still
haven't done it. Genuine tech debt, no excuse, on the list.

### 2. The sync queue driver: fast tests that lie a little

The message queue has swappable drivers —
`src/engine/core-modules/message-queue/drivers/bullmq.driver.ts` for production,
`sync.driver.ts` for tests and local dev without Redis. The sync driver runs jobs
**inline, immediately, in-process**.

Chosen because it makes job-dependent tests deterministic and fast: enqueue,
job runs, assert. No polling, no Redis in unit CI.

The lie: the sync driver erases every interesting property of a real queue.
No retries (a `retryLimit: 3` option is a no-op), no concurrency, no
at-least-once redelivery, no job loss on crash. **Any test that "proves" retry
or idempotency behavior under the sync driver proves nothing.** The webhook
delivery bug in doc 05 sailed through CI precisely because of this — the failure
mode only exists under BullMQ semantics. When you write a job, ask: "does my
correctness depend on queue semantics?" If yes, it needs an integration test
under the real driver, and you should assume the sync-driver test is
documentation, not verification.

### 3. Codegen'd test clients vs. hand-rolled request helpers

Integration tests talk to the GraphQL API through generated, typed clients
(`test/integration/graphql/codegen/`). Rejected: hand-written query strings with
`supertest`. The codegen approach means a schema change breaks tests at
*compile time* instead of at runtime-with-a-cryptic-null — with a runtime-generated
schema (doc 02), typed clients are one of the few places we get static confirmation
that the generated API still has the shape we promised.

Tradeoff: the codegen step is one more thing that goes stale. If your test fails
with a type error on a field you *know* exists, regenerate before you debug
(`npx nx run twenty-front:graphql:generate` for the front; the integration
codegen has its own target). New folks lose an afternoon to this roughly once.
Consider this paragraph your afternoon back.

## Evidence to go read

- `packages/twenty-server/test/integration/` — pick one `metadata/` suite and one
  `graphql/` suite; note how they create metadata and then query records through it
- `src/engine/core-modules/message-queue/drivers/sync.driver.ts` vs
  `bullmq.driver.ts` — diff their semantics mentally; this gap is a recurring exam topic
- `src/engine/workspace-manager/dev-seeder/` — what the seeded test workspace contains
- `packages/twenty-e2e-testing/` — the Playwright setup; note how few E2E tests
  there are relative to integration, and that this is on purpose

## A production incident this strategy prevented

A refactor of the migration builder (the doc 01 `flat-*` diff machinery) passed
every unit test — the diffing logic was pure and well-covered. The integration
suite caught that the generated DDL for a `RELATION` field's foreign key referenced
the wrong schema qualification for cross-object relations, something no unit test
would see because the bug lived in the *composition* of correct parts against real
Postgres name resolution. That's a "corrupt every new customer relation silently"
bug, stopped by a slow test tier that costs us real CI money. Every time someone
proposes mocking Postgres to speed up CI, I re-tell this story, and the proposal
dies. Cheap tests that don't test the risk aren't cheap.

## Before you move on, you should be able to answer:

1. Which properties of the production job system does `sync.driver.ts` not
   reproduce, and what concrete kind of bug does that let through CI?
2. Why do we insist on real Postgres for integration tests when it's the slowest
   part of CI? What did mocked-data-layer tests get us historically?
3. You've written a new metadata mutation. Which test tier(s) must cover it
   before I'll approve the PR, and what specifically should the integration test
   assert beyond "the mutation returned 200"?
