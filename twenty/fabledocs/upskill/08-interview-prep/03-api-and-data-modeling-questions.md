# API & Data Modeling Questions — 11 Cards

## Q1: How would you model multi-tenancy in Postgres?
Repo anchor: schema-per-workspace ([03-architecture/02](../03-architecture-and-patterns/02-data-model-and-persistence.md); scoped datasource at [common-base-query-runner.service.ts#L144-L153](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-base-query-runner.service.ts#L144-L153))
Junior: "add a tenantId column."
Mid adds: three options — row-scoping (cheap, leak-prone), schema-per-tenant (structural isolation, Twenty's choice), db-per-tenant (strongest, operationally heavy) — with one tradeoff each.
Senior includes: migration fan-out cost (DDL × tenants), noisy-neighbor blast radius, and how connection scoping kills classic IDOR structurally.

## Q2: How do you support user-defined custom fields?
Repo anchor: dynamic DDL via workspace migrations; metadata-driven schema ([Flow 6](../01-codebase-cartography/05-key-flows.md))
Mid adds: EAV vs JSONB vs real columns table, then defend real columns: indexes, types, relational integrity.
Senior includes: the coherence problem — Postgres schema, metadata rows, and schema caches must move in lockstep; version-keyed invalidation as the mechanism; partial-failure reconciliation as the open risk.

## Q3: REST vs GraphQL — how do you choose? Why might you offer both?
Repo anchor: both exist over one pipeline ([rest-api-core.controller.ts](../../../packages/twenty-server/src/engine/api/rest/core/controllers/rest-api-core.controller.ts) vs generated GraphQL)
Mid adds: GraphQL for flexible product UIs (the CRM's every view is a different projection); REST for integrators (curl-able, cacheable, Zapier-friendly). Both viable *cheaply* only because transport ≠ logic.
Senior includes: dual-surface parity as a testable contract (Ticket M9); GraphQL-specific risks (complexity attacks) and the repo's caps ([common-base-query-runner.service.ts#L397-L431](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-base-query-runner.service.ts#L397-L431)).

## Q4: Design cursor vs offset pagination.
Repo anchor: Relay-style connections ([object-records-to-graphql-connection.helper.ts](../../../packages/twenty-server/src/engine/api/graphql/graphql-query-runner/helpers/object-records-to-graphql-connection.helper.ts)); pagination edge-case suite ([composite-field-pagination.integration-spec.ts](../../../packages/twenty-server/test/integration/graphql/suites/composite-field-pagination.integration-spec.ts))
Mid adds: offset drifts under concurrent writes and gets slow at depth; cursors encode a stable sort position; composite-type sort keys complicate cursor comparison — there's a whole test suite for it.
Senior includes: cursors require a total order (tiebreak on id); the repo's row-level *filtering* (not post-filtering) is what keeps counts/pages truthful under permissions.

## Q5: How do you validate input in a layered API?
Repo anchor: the six-layer gauntlet ([03-architecture/03](../03-architecture-and-patterns/03-validation-auth-and-permissions.md) table)
Mid adds: client validation is UX, server is law; typed error codes as contract; operation-specific `validate()` hooks ([common-create-one-query-runner.service.ts#L92-L103](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-create-one-query-runner.service.ts#L92-L103)).
Senior includes: validation placement follows *trust boundaries*, not code convenience; metadata-driven coercion for dynamic schemas.

## Q6: How do you evolve a schema without downtime?
Repo anchor: fast vs slow instance commands with mandatory `down` ([CLAUDE.md](../../../CLAUDE.md); [03-architecture/02](../03-architecture-and-patterns/02-data-model-and-persistence.md))
Mid adds: expand/contract — add nullable, backfill, then constrain; separate schema-shape from data-movement migrations (exactly the fast/slow split).
Senior includes: per-tenant fan-out (a migration is N migrations here); rollback as tested deliverable; never rewriting shipped migrations.

## Q7: How do you prevent IDOR?
Repo anchor: structural scoping + grants + row predicates ([03-architecture/03](../03-architecture-and-patterns/03-validation-auth-and-permissions.md))
Junior: "check ownership in the handler."
Mid adds: the ordering — connection-level tenant scoping first, per-object grants second, row predicates third; enforcement in the data layer so no endpoint can forget.
Senior includes: joins as the sneaky path ([permissions.utils.ts#L279-L360](../../../packages/twenty-server/src/engine/twenty-orm/repository/permissions.utils.ts#L279-L360)); the audited bypass flag.

## Q8: Authn for a public API: sessions, JWTs, API keys?
Repo anchor: three principals through one guard ([jwt-auth.guard.ts#L23-L49](../../../packages/twenty-server/src/engine/guards/jwt-auth.guard.ts#L23-L49))
Mid adds: browser = short-lived access + refresh; integrations = API keys (revocable, throttled — [common-base-query-runner.service.ts#L350-L395](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-base-query-runner.service.ts#L350-L395)); one context shape downstream.
Senior includes: system/job principals ([buildSystemAuthContext](../../../packages/twenty-server/src/modules/workflow/workflow-runner/jobs/run-workflow.job.ts#L47)); metadata-version binding per request for cache coherence.

## Q9: Design webhooks for your API.
Repo anchor: the whole of [Flow 4](../01-codebase-cartography/05-key-flows.md) — signing, SSRF-safe egress, wildcards, chunking, at-most-once caveat
Mid adds: HMAC over timestamp+body, timeout, async via queue, audit both outcomes.
Senior includes: delivery-guarantee decision made explicitly (retries/backoff/DLQ/circuit breaker as the upgrade path — Project 2); payload as public contract with named dependents.

## Q10: Find and fix an N+1.
Repo anchor: centralized relation loading + complexity ban ([common-base-query-runner.service.ts#L406-L414](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-base-query-runner.service.ts#L406-L414)); dataloaders dir exists ([engine/dataloaders](../../../packages/twenty-server/src/engine/dataloaders))
Mid adds: detect via query logging (count queries per request); fix via join/batch/dataloader; the API-shape fix (forbidding pathological nesting) as the guardrail class.
Senior includes: N+1 across the *network* (front per-cell queries — Kata 3) is the same disease; fix at the layer that owns batching.

## Q11: Soft delete vs hard delete?
Repo anchor: `deleteOne`/`destroyOne` split; dedicated soft-delete builder; trash cleanup ([Pattern 13](../03-architecture-and-patterns/05-pattern-catalog.md))
Mid adds: soft = undo + audit; costs: every query filters `deletedAt`, uniqueness semantics change; scheduled purge for storage/privacy.
Senior includes: relations to soft-deleted rows (the repo tests this exact edge — [soft-deleted-relation.integration-spec.ts](../../../packages/twenty-server/test/integration/graphql/suites/soft-deleted-relation.integration-spec.ts)); legal deletion ≠ `deletedAt`.
