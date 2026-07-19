# Architecture Critique

An honest assessment, written the way you'd deliver it after three months owning this system. Confirmed observations are anchored; everything else is labeled hypothesis. This file doubles as system-design interview material — [08-interview-prep/04](../08-interview-prep/04-system-design-from-this-repo.md) builds directly on it.

## Strongest design choices

1. **Authorization enforced in the data layer.** Object grants, join validation, row predicates, and field restrictions all live in the query builders ([workspace-select-query-builder.ts#L342-L390](../../../packages/twenty-server/src/engine/twenty-orm/repository/workspace-select-query-builder.ts#L342-L390), [permissions.utils.ts#L279-L360](../../../packages/twenty-server/src/engine/twenty-orm/repository/permissions.utils.ts#L279-L360)). Feature velocity doesn't erode security because forgetting isn't possible. This is rare and worth imitating.
2. **One pipeline, many transports.** GraphQL, REST, and MCP all delegate to the common query runners. Validation/permission fixes are single-point ([common-base-query-runner.service.ts#L101-L159](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-base-query-runner.service.ts#L101-L159)).
3. **Dynamic DDL over EAV/JSONB** for custom fields — real columns keep indexes, types, and relational integrity ([02-data-model-and-persistence.md](02-data-model-and-persistence.md)). Costly to build, but it's the moat: query performance on custom objects matches standard ones.
4. **Version-keyed metadata caching.** Metadata version bumps invalidate schema SDL and flat maps coherently ([workspace-schema.factory.ts#L27-L31](../../../packages/twenty-server/src/engine/api/graphql/workspace-schema.factory.ts#L27-L31); version check per request at [jwt-auth.guard.ts#L29-L33](../../../packages/twenty-server/src/engine/guards/jwt-auth.guard.ts#L29-L33)).
5. **Security ergonomics:** centralized SSRF-safe egress client, HMAC-signed webhooks, standardized permission-denied messages, complexity limits on queries. Defaults are safe; effort is required to be unsafe.

## Risks and tradeoffs (prioritized)

| # | Risk | Evidence | Status |
| --- | --- | --- | --- |
| 1 | **No transactional outbox** — record events are in-process emits after SQL execution; crash or rollback windows can lose or fabricate events feeding webhooks/workflows/subscriptions | [workspace-insert-query-builder.ts#L255-L275](../../../packages/twenty-server/src/engine/twenty-orm/repository/workspace-insert-query-builder.ts#L255-L275), [workspace-event-emitter.ts#L39-L67](../../../packages/twenty-server/src/engine/workspace-event-emitter/workspace-event-emitter.ts#L39-L67) | Confirmed structure; loss frequency unquantified |
| 2 | **Webhook delivery is single-attempt at-most-once** — the catch swallows all failures so BullMQ retries never engage | [call-webhook.job.ts#L102-L116](../../../packages/twenty-server/src/engine/metadata-modules/webhook/jobs/call-webhook.job.ts#L102-L116) vs `retryLimit: 3` at [call-webhook-jobs.job.ts#L72](../../../packages/twenty-server/src/engine/metadata-modules/webhook/jobs/call-webhook-jobs.job.ts#L72) | Confirmed behavior; intent ambiguous — possibly deliberate |
| 3 | **Workflow run start race** — status checks are read-then-act; two workers could pass the guard concurrently | [run-workflow.job.ts#L90-L95](../../../packages/twenty-server/src/modules/workflow/workflow-runner/jobs/run-workflow.job.ts#L90-L95) | Hypothesis — depends on queue delivery + locking I did not verify |
| 4 | **Metadata migration partial failure** — DDL, metadata rows, and cache invalidation span systems; a crash mid-sequence could desync Postgres schema from metadata | [workspace-migration-runner](../../../packages/twenty-server/src/engine/workspace-manager/workspace-migration/workspace-migration-runner) | Investigate — transaction scope unread |
| 5 | **Base-runner accretion** — 13+ injected concerns and growing | [common-base-query-runner.service.ts#L64-L95](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-base-query-runner.service.ts#L64-L95) | Confirmed shape; cost is future maintainability |
| 6 | **Permission bypass flag** — correct today, but each new call site is reviewed only by convention | grep `shouldBypassPermissionChecks` | Confirmed mechanism; risk is process, not code |
| 7 | **User-token traffic unthrottled in pipeline** — only API keys hit the token bucket | [common-base-query-runner.service.ts#L352](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-base-query-runner.service.ts#L352) | Confirmed here; compensating controls elsewhere unverified |

## What I'd change owning this for 3 months

1. **Decide webhook retry semantics explicitly** (risk 2). Cheapest high-value fix: either rethrow on 5xx/timeout with capped backoff, or document at-most-once as contract. Migration: behavior flag per workspace; test: integration case with a failing receiver asserting attempt count. Blast radius: webhook consumers only.
2. **Add an outbox for queue-bound events** (risk 1). Write events to a table in the write transaction; a relay publishes to Redis. Keep in-process emits for live subscriptions. Migration path: dual-write with reconciliation metric, then cut over per queue. Test: kill the relay mid-batch, assert zero loss on restart.
3. **Compare-and-set on workflow run start** (risk 3): `UPDATE ... SET status='RUNNING' WHERE id=$1 AND status IN ('ENQUEUED','NOT_STARTED')` and proceed only if a row changed. Small diff, closes the race regardless of queue semantics; test with two concurrent starters.
4. **Bypass-flag audit rail** (risk 6): lint rule or wrapper requiring a reason string at each `shouldBypassPermissionChecks: true` site.
5. **Watch, don't touch:** the base runner (risk 5) until a concrete change hurts; refactors there are high-blast-radius with low user value today.

## Hypotheses I would test before acting

- Replica reads (read-only runners, [L332-L334](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-base-query-runner.service.ts#L332-L334)) causing user-visible staleness after writes — measure lag first.
- `refetchAggregateQueries` per create becoming a hotspot in bulk imports — profile before optimizing ([useCreateOneRecord.ts#L190-L192](../../../packages/twenty-front/src/modules/object-record/hooks/useCreateOneRecord.ts#L190-L192)).

## Interview usage

This critique is a template for the strongest possible answer to "tell me about a system you know well": three praised decisions with reasons, three risks with evidence, one prioritized fix with a migration and test plan. Practice delivering it in four minutes.
