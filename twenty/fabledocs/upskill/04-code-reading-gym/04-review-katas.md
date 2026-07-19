# Review Katas

Eight fake PRs. For each: read the intent and diff summary, write your review (findings graded **Blocking / Important / Optional**), then compare. All diffs are **illustrative fake code: not from this repo**, but each resembles real files (cited) so your review is grounded in this codebase's conventions.

Review language rule: every comment names the problem, the consequence, and a way forward. "This is wrong" is not a review comment.

---

## Kata 1: "Add retry to webhook delivery"

Author intent: retry failed webhook POSTs.
Fake diff summary: inside `callWebhook`'s catch block, adds `for (let i=0;i<3;i++) { try { await axiosClient.post(...); break; } catch {} }`.
Files this resembles: [call-webhook.job.ts#L102-L116](../../../packages/twenty-server/src/engine/metadata-modules/webhook/jobs/call-webhook.job.ts#L102-L116)
Expected findings — Blocking: immediate in-process retries hammer a failing receiver with no backoff; total job time can now hit 20s under `Promise.all` of 20 — worker starvation. Important: the queue already has retry machinery (`retryLimit` at enqueue + [bullmq.driver.ts#L248](../../../packages/twenty-server/src/engine/core-modules/message-queue/drivers/bullmq.driver.ts#L248)); rethrowing selectively is the idiomatic fix. Audit events now fire multiple times per logical delivery — is that intended? Optional: 4xx vs 5xx should be treated differently (4xx will never succeed).
Good review comment example:
> Retrying inline defeats the queue's backoff and blocks the chunk's other 19 deliveries. Could we instead rethrow on retryable statuses and let `retryLimit` handle spacing? That also keeps one audit event per attempt.

## Kata 2: "Fast count endpoint"

Author intent: add a total-count REST endpoint for dashboards.
Fake diff summary: new handler runs `repository.query('SELECT COUNT(*) FROM ' + tableName)` with `tableName` derived from the URL path.
Files this resembles: [rest-api-core.controller.ts](../../../packages/twenty-server/src/engine/api/rest/core/controllers/rest-api-core.controller.ts), handlers dir
Expected findings — Blocking: string-concatenated identifier = SQL injection vector even if values are parameterized; bypasses the permission-enforcing query builders entirely (raw query → no RLS predicates, no object grants — cross-user data leak via counts). Important: doesn't respect soft deletes (`deletedAt`). Optional: counts belong in the existing groupBy/aggregate pipeline; check [group-by-resolver.factory.ts](../../../packages/twenty-server/src/engine/api/graphql/workspace-resolver-builder/factories/group-by-resolver.factory.ts) precedent.

## Kata 3: "Show creator avatar in table"

Author intent: display `createdBy` avatar in the record table.
Fake diff summary: table cell component calls `useQuery(GET_USER, { variables: { id: record.createdById } })`.
Files this resembles: record-table cells under [modules/object-record](../../../packages/twenty-front/src/modules/object-record)
Expected findings — Blocking: per-cell query = N+1 over the network; 100 rows = 100 requests. Important: creator info is already in the record's `createdBy` composite/actor field or resolvable via depth-1 record gql fields ([useGenerateDepthRecordGqlFieldsFromObject](../../../packages/twenty-front/src/modules/object-record/graphql/record-gql-fields/hooks/useGenerateDepthRecordGqlFieldsFromObject.ts)) — extend the row query instead. Optional: memoize the cell; avatar component may exist in twenty-ui.

## Kata 4: "Quick fix: skip permission check for merge"

Author intent: merge-many was failing with permission errors for a support tooling script.
Fake diff summary: passes `shouldBypassPermissionChecks: true` into the merge path when the request has header `x-internal: true`.
Files this resembles: [common-merge-many-query-runner.service.ts](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-merge-many-query-runner.service.ts), bypass usage at [workspace-insert-query-builder.ts#L236](../../../packages/twenty-server/src/engine/twenty-orm/repository/workspace-insert-query-builder.ts#L236)
Expected findings — Blocking: a client-controlled header toggling permission bypass is privilege escalation, full stop. Important: legitimate system paths build a system auth context ([run-workflow.job.ts#L47](../../../packages/twenty-server/src/modules/workflow/workflow-runner/jobs/run-workflow.job.ts#L47)) — the script should authenticate as a principal with a proper role. Optional: if support tooling is a real need, propose an admin-scoped API surface with audit.

## Kata 5: "Debounce cell saves"

Author intent: reduce mutation spam while typing.
Fake diff summary: wraps `persistField` in `debounce(persist, 500)` inside the cell component; removes the deep-equality no-op check "since debounce handles it."
Files this resembles: [usePersistField.ts#L266-L268](../../../packages/twenty-front/src/modules/object-record/record-field/ui/hooks/usePersistField.ts#L266-L268)
Expected findings — Blocking: debounced saves + navigation away = lost final keystrokes unless flushed on blur/unmount. Important: the equality check prevented *no-op* mutations, a different problem than rapid-fire ones — removing it reintroduces junk writes (and junk `UPDATED` events feeding webhooks/workflows — blast radius beyond UI). Optional: repo convention already persists on commit-style events (Enter/blur), not per keystroke — is the problem real? Measure first.

## Kata 6: "Add `position` to webhook payload"

Author intent: consumers want record position in webhook payloads.
Fake diff summary: adds field to the payload object in `transformEventBatchToWebhookEvents`.
Files this resembles: [transform-event-batch-to-webhook-events.ts](../../../packages/twenty-server/src/engine/metadata-modules/webhook/utils/transform-event-batch-to-webhook-events.ts); contract warning at [call-webhook-jobs.job.ts#L34-L37](../../../packages/twenty-server/src/engine/metadata-modules/webhook/jobs/call-webhook-jobs.job.ts#L34-L37)
Expected findings — Blocking: none — additive payload fields are usually safe. Important: the code comment names two downstream dependents (Zapier trigger, OpenAPI webhook schema) that must be updated in the same PR; missing tests in the existing transform spec ([transform-event-batch-to-webhook-events.spec.ts](../../../packages/twenty-server/src/engine/metadata-modules/webhook/utils/__tests__/transform-event-batch-to-webhook-events.spec.ts)). Optional: does the HMAC signature docs/sample need regenerating?
Lesson: "small" diffs on public contracts are review-heavy by nature.

## Kata 7: "Cache user permissions in module scope"

Author intent: speed up permission resolution.
Fake diff summary: `const permissionCache = new Map()` at module top-level in a service; keyed by userWorkspaceId, no TTL, no invalidation on role change.
Files this resembles: [permissions.service.ts](../../../packages/twenty-server/src/engine/metadata-modules/permissions/permissions.service.ts)
Expected findings — Blocking: stale grants after role revocation = security bug (revoked user keeps access until process restart); multi-instance deployments each hold divergent caches. Important: the repo's caching idiom is version-keyed workspace cache (invalidated on metadata change) — use it. Optional: measure first; role maps may already be cached upstream (they're read from workspace context at [common-base-query-runner.service.ts#L316-L322](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-base-query-runner.service.ts#L316-L322)).

## Kata 8: "Tidy: merge createOne into createMany resolver"

Author intent: delete `CreateOneResolverFactory`, route createOne GraphQL field through the createMany factory with a length-1 array.
Files this resembles: [create-one-resolver.factory.ts](../../../packages/twenty-server/src/engine/api/graphql/workspace-resolver-builder/factories/create-one-resolver.factory.ts), [common-create-one-query-runner.service.ts#L36-L49](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-create-one-query-runner.service.ts#L36-L49)
Expected findings — Important: the *service* already delegates one→many (L40-L48); the resolver factories differ in response shape (single record vs list) and GraphQL field wiring — collapsing them changes the public schema (breaking change) unless response mapping is preserved. Missing: schema-diff evidence, integration test run. Optional: if the refactor is behavior-preserving, ask for the generated SDL diff attached to the PR. Blocking: only if schema changes — then it's a versioning conversation, not a tidy-up.
Lesson: "refactor" PRs touching generated public contracts need contract-level proof, not just green unit tests.

---

Self-grade per kata — Basic: caught the Blocking issue. Solid: findings correctly *tiered* (didn't block on style). Strong: at least one comment offers a concrete alternative anchored in an existing repo pattern, phrased kindly. Timed variant of katas 1/2/4 lives in [08-interview-prep/05](../08-interview-prep/05-debugging-and-code-review-rounds.md).
