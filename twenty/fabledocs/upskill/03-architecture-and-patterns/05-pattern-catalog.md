# Pattern Catalog

14 cards. Goal: recognition in the wild, not name-dropping. Each drill asks you to find or refactor something.

---

## Pattern 1: Template Method Pipeline

Problem it solves: many operations share a fixed sequence (throttle → validate → authorize → execute → enrich) but differ in the middle steps.
General shape: abstract base owns the sequence; subclasses implement declared abstract hooks.
Real example: [common-base-query-runner.service.ts#L101-L182](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-base-query-runner.service.ts#L101-L182) (`execute` fixed; `run/validate/computeArgs/processQueryResult` abstract)
Second example: [common-create-one-query-runner.service.ts#L36-L103](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-create-one-query-runner.service.ts#L36-L103)
Why this implementation works: security-critical steps can't be skipped by a forgetful subclass.
Failure modes: base class accretes concerns; hook ordering becomes implicit knowledge.
Use it when: sequence is invariant and safety-relevant. Avoid when: steps need reordering per case (use a composable middleware chain).
Interview angle: "How do you ensure every endpoint checks permissions?" — make the check structural.
Drill: list the exact order of side effects in `execute()` from memory, then verify.

## Pattern 2: Factory-Generated Resolvers (Code as Data)

Problem it solves: N objects × M operations would mean N×M hand-written resolvers.
General shape: a factory per operation returns a closure; a registry composes them into a schema.
Real example: [create-one-resolver.factory.ts#L28-L64](../../../packages/twenty-server/src/engine/api/graphql/workspace-resolver-builder/factories/create-one-resolver.factory.ts#L28-L64)
Second example: the registry — [workspace-schema.factory.ts#L48-L67](../../../packages/twenty-server/src/engine/api/graphql/workspace-schema.factory.ts#L48-L67)
Why this implementation works: one fix improves every object's API simultaneously.
Failure modes: debugging generated code (no file to open for `createOneCompany`); per-object special-casing gets awkward (that's what query hooks are for).
Use it when: operations are uniform across entities. Avoid when: each entity's behavior genuinely diverges.
Interview angle: "How would you build an API over user-defined models?"
Drill: find where `DELETE_ONE`'s factory differs from `DESTROY_ONE`'s and why both exist.

## Pattern 3: Permission-Enforcing Query Builder (Policy in the Data Layer)

Problem it solves: authorization scattered across resolvers gets forgotten.
General shape: subclass the ORM's query builder; validate the expression tree and inject predicates on every execute path.
Real example: [workspace-select-query-builder.ts#L342-L390](../../../packages/twenty-server/src/engine/twenty-orm/repository/workspace-select-query-builder.ts#L342-L390)
Second example: join validation in [permissions.utils.ts#L279-L360](../../../packages/twenty-server/src/engine/twenty-orm/repository/permissions.utils.ts#L279-L360)
Why this implementation works: row filtering happens *in SQL*, so pagination/counts stay truthful.
Failure modes: the bypass flag; new execute paths that forget to call `validatePermissions` (note how `executeExistsQuery` is blocked outright, L335-L340).
Use it when: multi-tenant or role-scoped data. Avoid when: single-tenant internal tools (cost > benefit).
Interview angle: "Where should row-level security live?" — compare with Postgres RLS as the alternative.
Drill: find one more execute variant in the file and confirm it validates first.

## Pattern 4: Ambient Context (AsyncLocalStorage)

Problem it solves: threading workspace/auth through 30 call layers.
General shape: enter a context scope at the boundary; read it anywhere below.
Real example: `executeInWorkspaceContext(...)` at [common-base-query-runner.service.ts#L144-L153](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-base-query-runner.service.ts#L144-L153); read via `getWorkspaceContext()` (L316)
Second example: request binding in [jwt-auth.guard.ts#L47](../../../packages/twenty-server/src/engine/guards/jwt-auth.guard.ts#L47)
Why this implementation works: eliminates parameter drilling for cross-cutting identity.
Failure modes: hidden dependency (functions look pure but aren't); context loss across raw emitters/timers.
Use it when: identity/tracing spans a whole request. Avoid when: explicit parameters are feasible and few.
Interview angle: "How does request context work in Node without thread-locals?"
Drill: explain why the webhook job payload carries `workspaceId` explicitly instead of relying on this.

## Pattern 5: Optimistic UI with Compensation

Problem it solves: waiting for server round-trips makes UIs feel dead.
General shape: apply predicted result locally → send mutation → reconcile or roll back.
Real example: [useCreateOneRecord.ts#L97-L188](../../../packages/twenty-front/src/modules/object-record/hooks/useCreateOneRecord.ts#L97-L188)
Second example: dual write in [usePersistField.ts#L270-L281](../../../packages/twenty-front/src/modules/object-record/record-field/ui/hooks/usePersistField.ts#L270-L281)
Why this implementation works: client-minted UUID (L87) means the optimistic record and server record are *the same identity* — reconciliation is a merge, not a swap.
Failure modes: forgotten rollback path; optimistic values diverging from server-computed ones.
Use it when: success is the overwhelmingly common case. Avoid when: server-side validation frequently rejects (roll-forward UIs lie).
Interview angle: "Design an optimistic update. What happens on failure?"
Drill: refactor a fake pessimistic save into this shape (see [04-code-reading-gym/03-fake-code-contrasts.md](../04-code-reading-gym/03-fake-code-contrasts.md), contrast 4).

## Pattern 6: Queue Abstraction with Pluggable Drivers

Problem it solves: coupling app code to BullMQ makes tests slow and local dev heavy.
General shape: an interface (`add`, `@Processor`, `@Process`) with BullMQ and synchronous in-process drivers.
Real example: [drivers/bullmq.driver.ts](../../../packages/twenty-server/src/engine/core-modules/message-queue/drivers/bullmq.driver.ts) vs [drivers/sync.driver.ts](../../../packages/twenty-server/src/engine/core-modules/message-queue/drivers/sync.driver.ts)
Second example: consumption via decorators, [call-webhook.job.ts#L15-L34](../../../packages/twenty-server/src/engine/metadata-modules/webhook/jobs/call-webhook.job.ts#L15-L34)
Why this implementation works: same job code runs inline in tests and distributed in prod.
Failure modes: abstractions hide driver-specific semantics — `retryLimit` means nothing if jobs swallow errors; sync driver runs jobs in-transaction context where prod wouldn't (subtle test-prod divergence).
Use it when: you need queues *and* fast tests. Avoid when: you depend on deep BullMQ features (flows, rate limiters) that the abstraction hides.
Interview angle: "How do you test code that enqueues jobs?"
Drill: find where the driver is chosen based on config.

## Pattern 7: Wildcard Event Subscription Matching

Problem it solves: letting consumers subscribe to "all creates" or "everything on company" without enumerating.
General shape: compute the set of patterns an event satisfies; match subscriptions against it.
Real example: [call-webhook-jobs.job.ts#L39-L59](../../../packages/twenty-server/src/engine/metadata-modules/webhook/jobs/call-webhook-jobs.job.ts#L39-L59) (`company.created` → also `*.created`, `company.*`, `*.*`)
Second example: No second example found (subscriptions module has related routing — verify before citing).
Why this implementation works: four string checks, no regex, O(subscriptions).
Failure modes: pattern vocabulary is a public contract (Zapier depends on it); adding a third segment breaks the split.
Interview angle: topic-based pub/sub routing in one function.
Drill: what happens for a hypothetical event named `company.fieldsUpdated.bulk`? Read the split at L39.

## Pattern 8: Flat Snapshot Caching (Metadata as Immutable Maps)

Problem it solves: every request needs metadata; hitting Postgres per request would melt it.
General shape: denormalize the object graph into id-keyed maps, cache per version, pass by reference.
Real example: `flatObjectMetadataMaps`/`flatFieldMetadataMaps` flowing through [common-base-query-runner.service.ts#L105-L110](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-base-query-runner.service.ts#L105-L110); recompute-or-fetch in [call-webhook-jobs.job.ts#L48-L51](../../../packages/twenty-server/src/engine/metadata-modules/webhook/jobs/call-webhook-jobs.job.ts#L48-L51) (`getOrRecompute`)
Second example: SDL cache, [workspace-schema.factory.ts#L27-L31](../../../packages/twenty-server/src/engine/api/graphql/workspace-schema.factory.ts#L27-L31)
Why this implementation works: version-keyed caches make invalidation a version bump, not a hunt.
Failure modes: stale version propagation; giant maps in memory per workspace.
Interview angle: "cache invalidation is hard" — versioning is the escape hatch.
Drill: find where the metadata version is bumped when a field is added (start in field-metadata service / workspace-cache).

## Pattern 9: Signed Webhooks (HMAC + timestamp + nonce)

Problem it solves: receivers must verify the payload came from Twenty and isn't replayed.
General shape: HMAC-SHA256 over `timestamp:body` with a shared secret; send signature + timestamp + nonce headers.
Real example: [call-webhook.job.ts#L23-L70](../../../packages/twenty-server/src/engine/metadata-modules/webhook/jobs/call-webhook.job.ts#L23-L70)
Second example: secret generation, [generate-webhook-secret.util.ts](../../../packages/twenty-server/src/engine/metadata-modules/webhook/utils/generate-webhook-secret.util.ts)
Why this implementation works: timestamp binding bounds replay windows; secret never leaves the payload builder (`const { secret, ...payloadWithoutSecret }`, L58).
Failure modes: receivers that don't verify; JSON key-order instability making signatures brittle across serializers.
Interview angle: "Secure a webhook" — this is the checklist answer, plus SSRF on the *sending* side (Pattern 10).
Drill: write receiver-side verification pseudocode including tolerance window.

## Pattern 10: Centralized Egress Guard (SSRF defense)

Problem it solves: user-supplied URLs (webhooks) can point at internal services/metadata endpoints.
General shape: one HTTP client factory that resolves and blocks private/internal IPs; all user-URL egress goes through it.
Real example: [call-webhook.job.ts#L72-L88](../../../packages/twenty-server/src/engine/metadata-modules/webhook/jobs/call-webhook.job.ts#L72-L88) using [secure-http-client.service.ts](../../../packages/twenty-server/src/engine/core-modules/secure-http-client/secure-http-client.service.ts); blocked-IP handling at L103-L114
Second example: search other `secureHttpClientService.getHttpClient` call sites (workflow HTTP steps — verify).
Why this implementation works: policy in one client beats checks sprinkled at call sites.
Failure modes: any user-URL fetch that *doesn't* use this client; DNS rebinding if resolution and connection are separate steps (*investigate* the client's internals before claiming either way).
Interview angle: "What's SSRF and how do you stop it?" — with a real implementation to cite.
Drill: grep for `axios` uses outside this service and classify each.

## Pattern 11: Status-Guarded State Machine Jobs

Problem it solves: queue redelivery and concurrent workers must not double-run work.
General shape: persist a status; each handler proceeds only from expected states.
Real example: [run-workflow.job.ts#L90-L95, L149-L151](../../../packages/twenty-server/src/modules/workflow/workflow-runner/jobs/run-workflow.job.ts#L90-L95)
Second example: staled-run cleanup command, [workflow-handle-staled-runs.command.ts](../../../packages/twenty-server/src/modules/workflow/workflow-runner/workflow-run-queue/command/workflow-handle-staled-runs.command.ts)
Why this implementation works: state in the DB survives crashes; the queue message is just a pointer.
Failure modes: read-then-act races without row locking; forgotten terminal states leaving "staled" runs (hence the cleanup command — the pattern's own patch).
Interview angle: "How do you make a job safe to run twice?"
Drill: draw the status transitions and mark which are compare-and-set vs read-then-write.

## Pattern 12: Versioned Immutable Definitions

Problem it solves: editing a workflow while runs are in flight.
General shape: definitions are versioned; executions pin a version id.
Real example: runs load their pinned `workflowVersionId` ([run-workflow.job.ts#L97-L101](../../../packages/twenty-server/src/modules/workflow/workflow-runner/jobs/run-workflow.job.ts#L97-L101))
Second example: metadata version keying schema caches (Pattern 8).
Why this implementation works: immutability converts a race into a non-event.
Failure modes: version sprawl (needs cleanup — see `number-of-workflow-runs-to-keep.ts`); "which version is live" UX complexity.
Interview angle: same trick as blue-green deploys and append-only schemas — say so.
Drill: find where a new workflow version is created on edit.

## Pattern 13: Soft Delete as a First-Class State

Problem it solves: users delete things by accident; support needs undo.
General shape: `deletedAt` timestamp; "delete" sets it, "destroy" removes the row; default reads filter it.
Real example: separate resolver factories [delete-one-resolver.factory.ts](../../../packages/twenty-server/src/engine/api/graphql/workspace-resolver-builder/factories/delete-one-resolver.factory.ts) vs [destroy-one-resolver.factory.ts](../../../packages/twenty-server/src/engine/api/graphql/workspace-resolver-builder/factories/destroy-one-resolver.factory.ts); dedicated [workspace-soft-delete-query-builder.ts](../../../packages/twenty-server/src/engine/twenty-orm/repository/workspace-soft-delete-query-builder.ts)
Second example: trash cleanup, [engine/trash-cleanup](../../../packages/twenty-server/src/engine/trash-cleanup)
Why this implementation works: restore + audit for free; permanent purge is scheduled, deliberate.
Failure modes: forgotten `deletedAt` filters (uniqueness constraints and joins get weird); "deleted" data still subject to privacy law.
Interview angle: "How would you implement undo for deletes?"
Drill: check how unique-ish behavior interacts with soft-deleted duplicates (find-duplicates runner is a good lens).

## Pattern 14: Fire-and-Forget Telemetry

Problem it solves: observability must never break or slow the business path.
General shape: `void promise` (or queue) for metrics/audit; failures tolerated silently.
Real example: [call-webhook.job.ts#L92-L101](../../../packages/twenty-server/src/engine/metadata-modules/webhook/jobs/call-webhook.job.ts#L92-L101)
Second example: metrics on throttle rejection, [common-base-query-runner.service.ts#L387-L394](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-base-query-runner.service.ts#L387-L394)
Why this implementation works: explicit `void` marks the intent; readers know it's deliberate.
Failure modes: applying the same shape to business writes; silent telemetry outages hiding incidents.
Interview angle: "What's allowed to fail silently?" — a surprisingly revealing question; have a rule.
Drill: state your rule in one sentence, then test it against three `void` call sites in the repo.
