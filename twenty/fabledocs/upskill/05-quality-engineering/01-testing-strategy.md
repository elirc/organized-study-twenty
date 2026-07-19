# Testing Strategy

## The layers as they exist here

| Layer | Where | Runs against | Speed | What belongs |
| --- | --- | --- | --- | --- |
| Unit (server) | `__tests__/` colocated, e.g. [merge-field-values.util.spec.ts](../../../packages/twenty-server/src/engine/api/graphql/graphql-query-runner/utils/__tests__/merge-field-values.util.spec.ts) | pure functions, mocked services | ms | transforms, guards, utils, decision logic |
| Unit (front) | colocated `__tests__` + Storybook stories | components, hooks with testing-library | ms–s | rendering logic, hook behavior |
| Integration (server) | [test/integration/](../../../packages/twenty-server/test/integration), `*.integration-spec.ts` | **real Postgres + Redis**, real GraphQL | s | resolver behavior, permissions, pagination, soft-delete semantics |
| E2E | [twenty-e2e-testing](../../../packages/twenty-e2e-testing) (Playwright) | full running app | min | critical user journeys only |

Target distribution per [CLAUDE.md](../../../CLAUDE.md): 70/20/10. The integration layer is where this repo's *real* risk lives (generated schema + permissions + SQL), which is why suites like [soft-deleted-relation.integration-spec.ts](../../../packages/twenty-server/test/integration/graphql/suites/soft-deleted-relation.integration-spec.ts) and [composite-field-pagination.integration-spec.ts](../../../packages/twenty-server/test/integration/graphql/suites/composite-field-pagination.integration-spec.ts) exist — each pins a bug-class, not a function.

## What belongs at which layer (decision rule)

Ask: *what can lie to me at this layer?* Unit tests can't catch SQL/permission/schema-generation lies — those need integration. Integration can't catch "the button doesn't fire" — E2E. If a test would pass with the database deleted, it's a unit test no matter what you mocked.

Anti-patterns to avoid here specifically:

- Unit-testing generated resolvers with mocked runners — you'd be testing the mock; the factory wiring is integration territory.
- E2E for permission matrices — combinatorial explosion; integration handles roles cheaply.
- Testing metadata *content* (does company have a name field) instead of metadata *behavior*.

## Fixtures, isolation, flake prevention

- Integration tests run with DB reset tooling (`test:integration:with-db-reset`, __inferred__ from CLAUDE.md) — determinism by construction.
- Test utilities live in [test/integration/](../../../packages/twenty-server/test/integration) (e.g. `metadata/utils/`) — always grep for an existing helper before writing raw GraphQL strings.
- Time and randomness: record IDs are uuids minted in code; assertions use helpers like [extract-record-ids-and-dates-as-expect-any.ts](../../../packages/twenty-server/test/utils/extract-record-ids-and-dates-as-expect-any.ts) — the repo's idiom for "assert shape, not nondeterministic values." Steal it.
- Front: `jest.clearAllMocks()` between tests, user-event over fireEvent, query by role/text ([CLAUDE.md](../../../CLAUDE.md) testing section).

## Commands (all __inferred__)

```bash
cd packages/twenty-server && npx jest "pattern"        # fastest loop, single file
npx nx test twenty-server                              # full unit suite
npx nx run twenty-server:test:integration:with-db-reset
npx nx test twenty-front
npx nx storybook:test twenty-front                     # story-based component tests
```

## Interview angle

"How do you decide what to test?" — answer with the *lie-detection* rule and the 70/20/10 shape, then give the soft-deleted-relation suite as an example of a regression test born from a bug class. "How do you deal with flaky tests?" — DB reset, shape-not-value assertions, no wall-clock sleeps. Cross-link: [08-interview-prep/05](../08-interview-prep/05-debugging-and-code-review-rounds.md).

Drill: open [all-people-resolvers.integration-spec.ts](../../../packages/twenty-server/test/integration/graphql/suites/all-people-resolvers.integration-spec.ts) and classify five of its assertions by which lie each would catch. Strong: you find one assertion you'd *delete* (testing incidental shape) and can defend that.
