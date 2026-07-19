# Writing Tests Here — Six Recipes

Each recipe: what you're pinning, where to put it, the repo idiom to copy, and the command (all commands __inferred__).

## Recipe 1 — Pure util (happy path + edge)

Pinning: a transform's contract. Copy the idiom of [transform-event-to-webhook-event.spec.ts](../../../packages/twenty-server/src/engine/metadata-modules/webhook/utils/__tests__/transform-event-to-webhook-event.spec.ts): colocated `__tests__/`, one `describe` per function, names like "should [behavior] when [condition]".

```bash
cd packages/twenty-server && npx jest transform-event-to-webhook-event
```

## Recipe 2 — Validation failure (server)

Pinning: bad input is rejected *with the right error code*, not a 500. Model: runner `validate()` methods throw typed exceptions (e.g. invalid UUID via [assert-is-valid-uuid](../../../packages/twenty-server/src/engine/api/graphql/workspace-query-runner/utils/assert-is-valid-uuid.util.ts), called at [common-create-one-query-runner.service.ts#L100-L102](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-create-one-query-runner.service.ts#L100-L102)). Unit-test the runner's `validate` directly with a fake context; assert exception class + code, not message text (messages are UX, codes are contract).

## Recipe 3 — Permission failure & cross-tenant rejection (integration)

Pinning: role without READ on object X gets denied; workspace A token cannot see workspace B rows. Put it in [test/integration/graphql/suites/](../../../packages/twenty-server/test/integration/graphql/suites) next to [auth.integration-spec.ts](../../../packages/twenty-server/test/integration/graphql/suites/auth.integration-spec.ts). Idiom: perform the operation twice — once as allowed role (expect data), once as denied (expect the standardized permission error; note the pipeline deliberately returns a generic message, [common-base-query-runner.service.ts#L304-L309](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-base-query-runner.service.ts#L304-L309)). The paired positive case is what makes the negative case trustworthy.

```bash
npx nx run twenty-server:test:integration:with-db-reset -- --testPathPattern=your-suite
```

## Recipe 4 — Async side effect (job behavior)

Pinning: webhook job semantics (e.g. "a 500 response records a failed audit event and does not throw"). Unit-test [CallWebhookJob](../../../packages/twenty-server/src/engine/metadata-modules/webhook/jobs/call-webhook.job.ts#L43-L117) with mocked `SecureHttpClientService`/`AuditService`: make the client reject, assert `insertWorkspaceEvent` called with `success: false` and the handler resolves. This is *exactly* the regression test the retry-semantics decision needs ([03-architecture-and-patterns/06](../03-architecture-and-patterns/06-architecture-critique.md), change 1) — if someone later makes it rethrow, this test forces the conversation.

## Recipe 5 — Soft delete / restore semantics (integration)

Pinning: deleted records vanish from findMany, survive in trash, restore intact, and *relations* respect `deletedAt`. Extend the pattern of [soft-deleted-relation.integration-spec.ts](../../../packages/twenty-server/test/integration/graphql/suites/soft-deleted-relation.integration-spec.ts). Use the shape-assertion helper [extract-record-ids-and-dates-as-expect-any.ts](../../../packages/twenty-server/test/utils/extract-record-ids-and-dates-as-expect-any.ts) instead of matching timestamps.

## Recipe 6 — Front hook with optimistic behavior

Pinning: `useCreateOneRecord` rolls back cache on mutation failure. Front idiom: renderHook with Apollo MockedProvider-style wrappers from [packages/twenty-front/src/testing/](../../../packages/twenty-front/src/testing) (mock hooks exist, e.g. [__mocks__/useCreateOneRecord.ts](../../../packages/twenty-front/src/modules/object-record/hooks/__mocks__/useCreateOneRecord.ts) — read it to see the established mock surface). Assert: cache contains optimistic record during flight, empty after rejection ([useCreateOneRecord.ts#L164-L188](../../../packages/twenty-front/src/modules/object-record/hooks/useCreateOneRecord.ts#L164-L188)).

```bash
cd packages/twenty-front && npx jest useCreateOneRecord
```

---

General rules for this repo: name the behavior, not the method; one behavior per test; prefer the integration layer whenever SQL/permissions/schema-generation is in the blast radius; never assert on generated IDs/timestamps directly. Before writing any helper, spend two minutes grepping `test/` for one that exists.

Drill: implement Recipe 4 for real. Self-grade — Basic: it passes. Solid: it fails meaningfully if the catch is removed. Strong: you added the mirrored case (2xx → `success: true`, metric incremented) and noticed the metric asymmetry between success and failure paths ([call-webhook.job.ts#L98-L101](../../../packages/twenty-server/src/engine/metadata-modules/webhook/jobs/call-webhook.job.ts#L98-L101) vs L108-L115) — worth a question to maintainers.
