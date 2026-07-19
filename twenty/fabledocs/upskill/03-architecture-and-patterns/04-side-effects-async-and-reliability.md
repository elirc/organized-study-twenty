# Side Effects, Async & Reliability

## Map of side effects

| Side effect | Trigger | Executor | Anchor |
| --- | --- | --- | --- |
| Webhook POSTs | record events | worker, `webhookQueue` | [call-webhook.job.ts](../../../packages/twenty-server/src/engine/metadata-modules/webhook/jobs/call-webhook.job.ts) |
| Workflow runs | DB event / cron / webhook / manual | worker, `workflowQueue` | [run-workflow.job.ts](../../../packages/twenty-server/src/modules/workflow/workflow-runner/jobs/run-workflow.job.ts) |
| Email/calendar sync | cron + connected accounts | worker | [modules/messaging/message-import-manager](../../../packages/twenty-server/src/modules/messaging/message-import-manager) |
| Emails (invites, verification) | user actions | worker via `twenty-emails` templates | [engine/core-modules/email](../../../packages/twenty-server/src/engine/core-modules/email) |
| Audit + metrics | everywhere | fire-and-forget `void` calls | [call-webhook.job.ts#L92-L101](../../../packages/twenty-server/src/engine/metadata-modules/webhook/jobs/call-webhook.job.ts#L92-L101) |
| Real-time subscriptions | record events | server, Redis pubsub | [engine/subscriptions](../../../packages/twenty-server/src/engine/subscriptions) |
| Front browser events | record ops | browser | [useCreateOneRecord.ts#L205-L211](../../../packages/twenty-front/src/modules/object-record/hooks/useCreateOneRecord.ts#L205-L211) |

## The event backbone — and its central caveat

Every ORM write emits a typed batch event in-process via EventEmitter2 ([workspace-insert-query-builder.ts#L255-L275](../../../packages/twenty-server/src/engine/twenty-orm/repository/workspace-insert-query-builder.ts#L255-L275) → [workspace-event-emitter.ts#L39-L67](../../../packages/twenty-server/src/engine/workspace-event-emitter/workspace-event-emitter.ts#L39-L67)). Listeners fan out to queues (webhooks, workflows, subscriptions, audit).

**There is no transactional outbox.** The chain is: SQL write → in-process emit → listener enqueues to Redis. Two failure windows follow directly from the code:

1. Process crashes after the SQL write but before listeners enqueue → the write persisted, downstream consumers never hear about it (lost event).
2. If a caller wraps the write in a transaction that later rolls back, the event has already fired (phantom event). Whether such wrapping occurs on user paths needs verification per flow — label it *investigate*, but the *structural* exposure is real and is exactly what an outbox (events written to a DB table in the same transaction, relayed separately) eliminates.

This is the single best architecture-discussion topic in the repo: the team traded delivery guarantees for simplicity and lower write latency, with audit logs as the compensating observability. In an interview, present the tradeoff, not a bug report.

## Delivery guarantees, compared within one repo

| Property | Webhooks | Workflows |
| --- | --- | --- |
| Retry on failure | **No** — errors swallowed in-job ([call-webhook.job.ts#L102-L116](../../../packages/twenty-server/src/engine/metadata-modules/webhook/jobs/call-webhook.job.ts#L102-L116)); `retryLimit: 3` at enqueue ([call-webhook-jobs.job.ts#L72](../../../packages/twenty-server/src/engine/metadata-modules/webhook/jobs/call-webhook-jobs.job.ts#L72), mapped to BullMQ `attempts = 1 + retryLimit`, [bullmq.driver.ts#L248](../../../packages/twenty-server/src/engine/core-modules/message-queue/drivers/bullmq.driver.ts#L248)) never fires because the job never throws | **Yes-ish** — errors mark run FAILED then rethrow so the queue records failure ([run-workflow.job.ts#L63-L73](../../../packages/twenty-server/src/modules/workflow/workflow-runner/jobs/run-workflow.job.ts#L63-L73)) |
| Effective semantics | at-most-once, single attempt, 5s timeout | at-least-once attempt with resume-from-step |
| Re-entry safety | n/a (stateless POST) | status guards: only ENQUEUED/NOT_STARTED start, only RUNNING resume ([L90-L95, L149-L151](../../../packages/twenty-server/src/modules/workflow/workflow-runner/jobs/run-workflow.job.ts#L90-L95)) — read-then-act, so a two-worker race is theoretically possible (*investigate* locking) |
| Failure visibility | audit event per attempt, success or fail | run status + error message persisted |

**Idempotency** inventory: client-minted UUIDs make record creation *replayable in principle* ([useCreateOneRecord.ts#L87](../../../packages/twenty-front/src/modules/object-record/hooks/useCreateOneRecord.ts#L87)) — retrying the same input with the same id can't double-insert (PK conflict), though there's no explicit idempotency-key API. Workflow status guards give *effective* idempotency for job re-delivery. Webhook receivers get timestamp+signature+nonce headers ([call-webhook.job.ts#L60-L70](../../../packages/twenty-server/src/engine/metadata-modules/webhook/jobs/call-webhook.job.ts#L60-L70)) so *they* can dedupe — idempotency delegated across the contract.

## Timeouts, backpressure, chunking

- Outbound webhook timeout: 5s hard ([call-webhook.job.ts#L86](../../../packages/twenty-server/src/engine/metadata-modules/webhook/jobs/call-webhook.job.ts#L86)). Every network call without a timeout is a latent outage; this repo sets one — check any HTTP call you add.
- Fan-out chunked to 20 per job ([call-webhook-jobs.job.ts#L19,L66](../../../packages/twenty-server/src/engine/metadata-modules/webhook/jobs/call-webhook-jobs.job.ts#L19)) — bounds per-job memory/time and spreads load across workers. Chunk-size choice is a backpressure lever.
- API-key traffic throttled with token buckets, short + long windows ([common-base-query-runner.service.ts#L350-L395](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-base-query-runner.service.ts#L350-L395)) — two windows catch both bursts and sustained abuse.
- The queue abstraction has a **sync driver** ([sync.driver.ts](../../../packages/twenty-server/src/engine/core-modules/message-queue/drivers/sync.driver.ts)) that executes jobs inline — great for dev/tests, and a reminder that "queued" is a deployment decision, not a code property.

## Side effects in risky places (watch list)

- Event emission inside the ORM write path (above) — powerful, but couples data layer to messaging.
- `refetchAggregateQueries` after every create ([useCreateOneRecord.ts#L190-L192](../../../packages/twenty-front/src/modules/object-record/hooks/useCreateOneRecord.ts#L190-L192)) — a network side effect inside a hook; fine at human scale, hostile to bulk operations (possible risk, measure before optimizing).

## Drills

1. Design the minimal transactional outbox for record events here: table shape, writer, relay, ordering key, cleanup. Strong: you keep the in-process emitter for subscriptions (latency) and use the outbox only for queue-bound consumers, and you can say why.
2. An admin reports "my webhook fired twice for one update." Using the map above, list every point that could duplicate. Strong: you include BullMQ redelivery of `CallWebhookJobsJob` itself (its handler *is* retried — `retryLimit: 3` — and enqueues in a loop; partial failure between chunks re-enqueues earlier chunks — trace it).

## Interview angle

At-least-once vs at-most-once, outbox, idempotency keys, retry budgets, timeout discipline — all anchored here. This file plus [Flow 4](../01-codebase-cartography/05-key-flows.md#flow-4-record-event--webhook-delivery-async-boundary) is your prep for any "design notifications/webhooks/jobs" question. ([08-interview-prep/04](../08-interview-prep/04-system-design-from-this-repo.md), variation 3)
