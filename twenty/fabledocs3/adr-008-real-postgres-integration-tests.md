# ADR-008: Integration tests against real Postgres, typed codegen'd clients, and a swappable sync queue driver

## Status

Accepted — with a known, unresolved blind spot: the sync queue driver erases
production queue semantics, so queue-dependent correctness is only provable in
the (thin) real-driver tier.

## Context

The riskiest code in this system generates SQL DDL and dynamic queries
(ADRs 002–005). Mocks of a database cannot reproduce Postgres's DDL semantics,
constraint behavior, name resolution, or error codes — a mock faithful enough
to be useful would *be* Postgres. Meanwhile thousands of unit tests need to
run fast, without Redis or a queue broker.

## Decision

A three-tier strategy with an explicit pyramid (CLAUDE.md states 70/20/10):

1. **Unit tests** — Jest, colocated `__tests__/` and `*.spec.ts`; single-file
   runs are the sanctioned dev loop.
2. **Integration tests against real Postgres** —
   `packages/twenty-server/test/integration/` (suites for `graphql/`,
   `metadata/`, `rest/`, `billing/`, `auth`…), run via
   `test:integration:with-db-reset`, seeded by
   `src/engine/workspace-manager/dev-seeder/`. Tests call the API through
   **generated typed clients** (`test/integration/graphql/codegen/`), so a
   schema change breaks tests at compile time, not as a runtime null.
3. **E2E** — `packages/twenty-e2e-testing/` (Playwright), deliberately thin.

Supporting decision: the message queue is a **driver interface**
(`src/engine/core-modules/message-queue/drivers/bullmq.driver.ts` vs
`sync.driver.ts`); tests and Redis-less dev run the sync driver, which
executes jobs inline and immediately.

## Alternatives considered

- **Mocked data layer / in-memory Postgres substitutes** — rejected (and the
  codebase's structure implies it was learned, not assumed): passing tests
  over a fake database verify nothing about the DDL engine, which is where
  the catastrophic bugs live.
- **Per-test database isolation** — not implemented; reset is per-run
  (`with-db-reset`), accepting intra-suite data coupling as the price of
  speed. Notably, ADR-003 makes per-test *workspace* isolation cheap in
  principle; its absence reads as backlog, not philosophy.
- **Running all tests under real BullMQ** — rejected for determinism and CI
  cost; accepted consequence is the sync-driver blind spot below.

## Consequences

**Positive:** the migration/query engines are tested against the only oracle
that matters; typed test clients turn API drift into compile errors —
valuable precisely because the schema is runtime-generated (ADR-006); unit
CI needs no Redis.

**Negative:** integration CI is slow and Postgres-stateful (test-order
couplings occur); codegen'd clients go stale and cost newcomers an afternoon;
and the sharp edge — `sync.driver.ts` provides **no retries, no concurrency,
no redelivery, no crash-loss**, so any test "verifying" retry or idempotency
behavior under it is vacuous. Queue-semantics bugs (ADR-010's webhook
delivery gap being the canonical example) pass this suite silently.

## Evidence

`packages/twenty-server/test/integration/` layout and
`test/integration/graphql/codegen/`;
`src/engine/core-modules/message-queue/drivers/sync.driver.ts` vs
`bullmq.driver.ts`; `src/engine/workspace-manager/dev-seeder/`; CLAUDE.md's
testing commands and pyramid; `packages/twenty-e2e-testing/`.
