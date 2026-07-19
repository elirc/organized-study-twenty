# Key Flows

Six end-to-end flows, rotated across UI, API, persistence, async, auth, and metadata. Later modules reference these — read them with the files open.

---

## Flow 1: Create a record (UI → GraphQL → Postgres)

Why this flow matters: it's the canonical write path; every layer of the architecture participates once.

Open these files first:
- [useCreateOneRecord.ts](../../../packages/twenty-front/src/modules/object-record/hooks/useCreateOneRecord.ts#L84-L220) — optimistic client
- [create-one-resolver.factory.ts](../../../packages/twenty-server/src/engine/api/graphql/workspace-resolver-builder/factories/create-one-resolver.factory.ts#L28-L64) — generated resolver
- [common-base-query-runner.service.ts](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-base-query-runner.service.ts#L101-L159) — pipeline
- [workspace-insert-query-builder.ts](../../../packages/twenty-server/src/engine/twenty-orm/repository/workspace-insert-query-builder.ts#L222-L275) — SQL + events

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | front | useCreateOneRecord.ts#L87 | Client mints `id = v4()` | `Partial<ObjectRecord>` | duplicate submit reuses new ids — no dedupe |
| 2 | front | useCreateOneRecord.ts#L89-L113 | Sanitize input, compute optimistic record, write to Apollo cache | GQL record node | optimistic record diverges from server-computed fields |
| 3 | front | useCreateOneRecord.ts#L140-L163 | `mutate` with `update` callback re-running optimistic effects with server data | mutation payload | — |
| 4 | server | create-one-resolver.factory.ts#L33-L45 | Resolver closure parses `graphqlFields(info)`, builds runner context, delegates | `CreateOneResolverArgs` | — |
| 5 | server | common-base-query-runner.service.ts#L112-L121 | Throttle (API-key only) → `validate()` → settings-permission check for system objects | args + auth context | user-token traffic is *not* rate-limited here |
| 6 | server | common-create-one-query-runner.service.ts#L92-L103 | UUID validity, mutation-on-remote-object rejection | — | — |
| 7 | server | common-base-query-runner.service.ts#L196-L214 | `computeArgs` (field coercion) then **pre-query hooks** | coerced input | hooks are per-object hidden behavior |
| 8 | server | common-base-query-runner.service.ts#L313-L348 | Resolve role permission config; get permission-scoped repository | repository | throws INVALID_AUTH_CONTEXT if role map missing |
| 9 | orm | workspace-insert-query-builder.ts#L222-L224 | Validate RLS predicates for insert, run SQL | SQL INSERT | — |
| 10 | orm | workspace-insert-query-builder.ts#L229-L246 | Re-select inserted rows **with permission bypass** for event payloads | entities | bypass is intentional but a sharp tool |
| 11 | orm | workspace-insert-query-builder.ts#L255-L275 | Emit `CREATED` + `UPSERTED` batch events in-process | `DatabaseBatchEvent` | see Flow 4: events are not transactional |
| 12 | front | useCreateOneRecord.ts#L164-L188 | On error: delete optimistic record from cache (rollback) | — | — |
| 13 | front | useCreateOneRecord.ts#L190-L192 | Refetch aggregate queries | network refetch | one extra round-trip per create |

Validation and authorization: input validation at steps 5–7; object/settings permissions at [common-base-query-runner.service.ts#L274-L311](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-base-query-runner.service.ts#L274-L311); row/field permissions inside the query builder (step 9).

Persistence and side effects: single INSERT plus post-insert re-select; events fan out to webhooks/workflows/subscriptions (Flows 4, 5).

Tests that cover it: integration suites under [test/integration/graphql/suites/](../../../packages/twenty-server/test/integration/graphql/suites) (e.g. `all-people-resolvers.integration-spec.ts`); front hook logic has util-level tests but the full optimistic path is exercised mainly via Playwright E2E.

What juniors usually miss: the record ID is chosen by the *client*; the "optimistic" record is a fabrication that must be reverted on failure.

What seniors notice: createOne delegates to createMany (one code path to harden); events are emitted after `super.execute()` but whether a caller wraps this in a transaction determines if events can fire for rolled-back writes — *investigate before asserting*.

Interview angle: "How do you build an optimistic UI that doesn't lie to users?" — answer with steps 1, 2, 12.

Drill: predict what happens if two rapid duplicate submits race. Then find the code that would or wouldn't dedupe them. Self-grade — Basic: traced both submits. Solid: identified client-side `loading` state as the only brake. Strong: proposed an idempotency key design and named its storage/TTL tradeoffs.

---

## Flow 2: findMany with filters (read path + permissions)

Why this flow matters: reads dominate traffic, and this is where tenant isolation and query-complexity limits live.

Open these files first:
- [find-many-resolver.factory.ts](../../../packages/twenty-server/src/engine/api/graphql/workspace-resolver-builder/factories/find-many-resolver.factory.ts) — entry
- [common-find-many-query-runner.service.ts](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-find-many-query-runner.service.ts) — runner
- [graphql-query.parser.ts](../../../packages/twenty-server/src/engine/api/graphql/graphql-query-runner/graphql-query-parsers/graphql-query.parser.ts) — filter/order parsing
- [workspace-select-query-builder.ts](../../../packages/twenty-server/src/engine/twenty-orm/repository/workspace-select-query-builder.ts#L335-L390) — enforcement

Trace (condensed): resolver → base pipeline (same as Flow 1 steps 5–8) → parser converts GraphQL filter tree into TypeORM query → `WorkspaceSelectQueryBuilder` calls `validatePermissions()` before every execute variant ([L342-L352](../../../packages/twenty-server/src/engine/twenty-orm/repository/workspace-select-query-builder.ts#L342-L352)): applies row-level permission predicates ([L369-L390](../../../packages/twenty-server/src/engine/twenty-orm/repository/workspace-select-query-builder.ts#L369-L390)) then validates the expression map against object permissions, including joins ([permissions.utils.ts#L279-L360](../../../packages/twenty-server/src/engine/twenty-orm/repository/permissions.utils.ts#L279-L360)) → results shaped into a Relay-style connection ([object-records-to-graphql-connection.helper.ts](../../../packages/twenty-server/src/engine/api/graphql/graphql-query-runner/helpers/object-records-to-graphql-connection.helper.ts)).

Guardrails worth quoting in interviews: nested one-to-many inside one-to-many is flat-out rejected, and total relation complexity is capped by config ([common-base-query-runner.service.ts#L397-L431](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-base-query-runner.service.ts#L397-L431)). Read replicas: read-only runners request a replica datasource ([L332-L334](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-base-query-runner.service.ts#L332-L334)) — a consistency/latency tradeoff (replica lag can return stale reads).

Tests that cover it: [order-by-relation-field.integration-spec.ts](../../../packages/twenty-server/test/integration/graphql/suites/order-by-relation-field.integration-spec.ts), `composite-field-pagination.integration-spec.ts`, and friends.

What juniors miss: permissions are checked *twice at different granularities* — settings-level in the pipeline, row/field-level in the ORM. What seniors notice: enforcement at the lowest layer means even feature code that bypasses the resolver still can't leak rows — defense in depth with a deliberate `shouldBypassPermissionChecks` escape hatch that must be audited wherever it appears (e.g. [workspace-insert-query-builder.ts#L236](../../../packages/twenty-server/src/engine/twenty-orm/repository/workspace-insert-query-builder.ts#L236)).

Interview angle: "Where should authorization live — middleware, service, or data layer?" Use this two-tier design as your concrete example.

Drill: write the GraphQL query that would trip the nested one-to-many guard, and explain what DB behavior the guard prevents. Self-grade — Strong answer names cartesian explosion / unbounded fan-out, not just "it's slow."

---

## Flow 3: Login → authenticated request (auth boundary)

Why this flow matters: every other flow assumes this one worked; interviews love auth.

Open these files first:
- [auth service dir](../../../packages/twenty-server/src/engine/core-modules/auth) — token issuance (access + refresh; also API keys)
- [jwt-auth.guard.ts](../../../packages/twenty-server/src/engine/guards/jwt-auth.guard.ts#L23-L49) — request authentication
- [workspace-auth.guard.ts](../../../packages/twenty-server/src/engine/guards/workspace-auth.guard.ts#L12-L28) — workspace presence check

Trace:

| Step | Owner | File | What happens | Risk |
| --- | --- | --- | --- | --- |
| 1 | server | core-modules/auth (services, strategies) | Credentials or SSO (Google/Microsoft/OIDC/SAML guards in [auth/guards/](../../../packages/twenty-server/src/engine/core-modules/auth/guards)) verified; JWT access token issued | many providers = wide attack surface, each behind its own guard |
| 2 | front | Apollo link layer ([modules/apollo/](../../../packages/twenty-front/src/modules/apollo)) | Token attached to every GraphQL request | token refresh races |
| 3 | server | jwt-auth.guard.ts#L27-L33 | `validateTokenByRequest` → auth context; metadata version fetched for cache coherence | — |
| 4 | server | jwt-auth.guard.ts#L35-L45 | Reject if context has none of apiKey / userWorkspaceId / application — three principal types, one context shape | forgetting non-user principals when adding features |
| 5 | server | jwt-auth.guard.ts#L47 | `bindDataToRequestObject` — request now carries identity + workspace | request-scoped state; keep it read-only |
| 6 | server | workspace-auth.guard.ts#L22-L26 | Downstream guard just asserts `request.workspace` exists | returning `false` yields 403; distinguishing 401 vs 403 is a nuance worth knowing |

What juniors miss: "logged in" isn't one thing — user tokens, API keys, and applications all flow through the same guard and diverge later (e.g. only API keys are throttled in [common-base-query-runner.service.ts#L352](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-base-query-runner.service.ts#L352)).

What seniors notice: guards are small and composable; identity resolution happens once and is *bound to the request*, so downstream code never re-parses tokens — single point of truth, single point of failure.

Interview angle: "Access vs refresh tokens; where do you store them; how do you support API keys and OAuth in one API?" Anchor to steps 3–5.

Drill: list every guard in [engine/guards/](../../../packages/twenty-server/src/engine/guards) and classify: authentication, authorization, or environment gate. Self-grade — Strong: you can say which combination protects a settings mutation.

---

## Flow 4: Record event → webhook delivery (async boundary)

Why this flow matters: it's the repo's clearest reliability case study — signing, SSRF defense, retries, and an explicit at-most-once tradeoff, all in ~190 lines.

Open these files first:
- [workspace-event-emitter.ts](../../../packages/twenty-server/src/engine/workspace-event-emitter/workspace-event-emitter.ts#L39-L67) — in-process batch events
- [call-webhook-jobs.job.ts](../../../packages/twenty-server/src/engine/metadata-modules/webhook/jobs/call-webhook-jobs.job.ts#L30-L75) — fan-out
- [call-webhook.job.ts](../../../packages/twenty-server/src/engine/metadata-modules/webhook/jobs/call-webhook.job.ts#L43-L117) — delivery

Trace:

| Step | Owner | File | What happens | Risk |
| --- | --- | --- | --- | --- |
| 1 | orm | workspace-insert-query-builder.ts#L255-L275 | After INSERT, emit `company.created` batch via EventEmitter2 | in-process only: crash here = event lost (no outbox) |
| 2 | worker | call-webhook-jobs.job.ts#L39-L46 | Build match set: `company.created`, `*.created`, `company.*`, `*.*` | wildcard semantics are a contract; Zapier depends on payload (comment L34-L37) |
| 3 | worker | call-webhook-jobs.job.ts#L48-L59 | Load workspace's webhooks from cache, filter by operations | stale cache after webhook edits — cache invalidation dependency |
| 4 | worker | call-webhook-jobs.job.ts#L66-L74 | Chunk into 20s, enqueue `CallWebhookJob` with `retryLimit: 3` | retryLimit only helps if the job *throws* |
| 5 | worker | call-webhook.job.ts#L54-L70 | Headers: timestamp, HMAC-SHA256 signature over `timestamp:payload`, nonce | receiver must verify; replay protection needs receiver cooperation |
| 6 | worker | call-webhook.job.ts#L72-L88 | POST via `SecureHttpClientService` (SSRF-blocking), 5s timeout | 5s is generous per-call; chunk of 20 runs in `Promise.all` |
| 7 | worker | call-webhook.job.ts#L90-L116 | Audit event recorded for success *and* failure; **catch swallows the error** | failed delivery is never retried — the catch never rethrows, so `retryLimit: 3` never fires for HTTP failures |

Persistence and side effects: none in the DB beyond audit/metrics — that's the point: delivery state lives in the audit log, not a delivery table.

Tests that cover it: unit tests for payload transforms ([transform-event-to-webhook-event.spec.ts](../../../packages/twenty-server/src/engine/metadata-modules/webhook/utils/__tests__/transform-event-to-webhook-event.spec.ts)); no direct coverage found for the swallow-vs-retry behavior — good regression-test candidate.

What juniors miss: `retryLimit: 3` looks like "retries 3 times" but the inner catch makes delivery effectively **at-most-once with a single attempt**. Whether that's a bug or a product decision (avoid hammering broken receivers; failures are auditable) is a design conversation — treat it as *confirmed behavior, debatable intent*.

What seniors notice: no transactional outbox between step 1 and 2; SSRF defense is centralized in a client service instead of scattered checks (strength); nonce without receiver-side storage doesn't actually prevent replay (limitation to discuss, not a vuln claim).

Interview angle: "Design webhook delivery" — this flow *is* the model answer, including what you'd add (outbox, exponential backoff, dead-letter queue, per-endpoint circuit breaker).

Drill: write the regression test description (not code) that would lock in intended retry semantics either way. Self-grade — Strong: your test forces the team to *decide* the intent explicitly.

---

## Flow 5: Workflow run (background execution with resume)

Why this flow matters: long-running, multi-step background execution with crash-resume is a system-design staple.

Open these files first:
- [run-workflow.job.ts](../../../packages/twenty-server/src/modules/workflow/workflow-runner/jobs/run-workflow.job.ts#L38-L133) — job entry
- [workflow-executor.workspace-service.ts](../../../packages/twenty-server/src/modules/workflow/workflow-executor/workspace-services/workflow-executor.workspace-service.ts) — step engine

Trace (condensed): trigger (DB event / cron / webhook / manual — enumerated at [run-workflow.job.ts#L222-L237](../../../packages/twenty-server/src/modules/workflow/workflow-runner/jobs/run-workflow.job.ts#L222-L237)) enqueues a run → job builds a **system auth context** for the workspace (L47 — background work has no user; it must fabricate a principal) → status guard: only ENQUEUED/NOT_STARTED runs start (L90-L95), only RUNNING runs resume (L149-L151) — this makes duplicate deliveries of the same job *mostly* harmless (re-entry protection via state check; note it's a read-then-act check, not a DB-level lock — *investigate* whether a race between two workers is possible) → steps execute; resume path recomputes next steps from persisted run state (L134-L211) → any error marks the run FAILED and **rethrows** (L63-L73) so the queue sees the failure — the exact opposite choice from the webhook job, and a great compare/contrast.

What juniors miss: `lastExecutedStepId` in the job data is what makes resume possible — state lives in the DB (`workflowRun.state`), not in the queue message.

What seniors notice: versioned workflows (runs pin a `workflowVersionId`, L97-L101) prevent in-flight runs from being corrupted by edits — immutability as a consistency strategy.

Interview angle: "How do you resume a multi-step job after a crash?" — persisted step state + status guards + idempotent step design.

Drill: diagram the state machine of `WorkflowRunStatus` from the code you can find. Self-grade — Strong: you mark which transitions are guarded and which component performs each.

---

## Flow 6: Add a custom field (metadata → live schema)

Why this flow matters: this is Twenty's defining capability — schema changes at runtime, no deploy.

Open these files first:
- [field-metadata.service.ts](../../../packages/twenty-server/src/engine/metadata-modules/field-metadata/services/field-metadata.service.ts) — orchestrator
- [workspace-migration dir](../../../packages/twenty-server/src/engine/workspace-manager/workspace-migration) — DDL generation/execution
- [workspace-schema.factory.ts](../../../packages/twenty-server/src/engine/api/graphql/workspace-schema.factory.ts#L23-L68) — schema rebuild

Trace (condensed): settings UI calls the `/metadata` GraphQL API → field metadata validated and persisted → a workspace migration (typed DDL actions, e.g. add column) is built by the migration builder and executed by action handlers ([workspace-migration-runner/action-handlers/](../../../packages/twenty-server/src/engine/workspace-manager/workspace-migration/workspace-migration-runner/action-handlers)) → metadata caches invalidated; the metadata *version* bumps (the JWT guard fetches it per request at [jwt-auth.guard.ts#L29-L33](../../../packages/twenty-server/src/engine/guards/jwt-auth.guard.ts#L29-L33)) → next `/graphql` request finds no cached SDL for the new version and recomputes it via `getOrComputeSchemaSDL` ([workspace-schema.factory.ts#L27-L31](../../../packages/twenty-server/src/engine/api/graphql/workspace-schema.factory.ts#L27-L31)) → the front refetches object metadata and its generic record components just render the new field.

What juniors miss: there are *two* schemas changing in lockstep — the Postgres schema (DDL) and the GraphQL schema (cache recompute) — and a version number is the coherence mechanism between them.

What seniors notice: DDL + metadata row + cache invalidation must not partially fail; look for transaction/compensation handling in the migration runner before trusting it (*investigate* — don't assume). This is the highest-blast-radius subsystem in the repo: a bug here corrupts tenants' schemas.

Interview angle: "Design a system where users define custom fields" — the classic EAV vs JSONB vs dynamic-DDL question. Twenty chose dynamic DDL with real columns; be ready to defend and attack that choice ([03-architecture-and-patterns/02-data-model-and-persistence.md](../03-architecture-and-patterns/02-data-model-and-persistence.md)).

Drill: list the failure points if the process crashes between DDL execution and cache invalidation, and what a user would observe. Self-grade — Strong: you propose a detection mechanism (schema/metadata reconciliation check), not just a fix.
