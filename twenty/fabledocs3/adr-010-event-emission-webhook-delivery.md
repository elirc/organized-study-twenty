# ADR-010: In-process event emission with queue fan-out — and (implicitly) at-most-once webhook delivery

## Status

**Implicitly accepted — should be revisited.** This is the one ADR whose
"Decision" the authors may not know they made: the code contains contradictory
signals (a retry limit that cannot fire), which is the signature of an
accidental decision rather than a deliberate one.

## Context

Record mutations must fan out to consumers: webhooks, workflows, audit/
timeline, search indexing. This is textbook distributed-systems territory:
the mutation commits in Postgres, consumers are notified asynchronously, and
the guarantees (at-most-once vs at-least-once, ordering, durability across
crashes) must be *chosen*, because no choice is still a choice.

## Decision (as the code actually behaves)

1. **Emission**: after a mutation executes, an event batch is emitted
   **in-process** via `src/engine/workspace-event-emitter/`
   (`WorkspaceEventBatch` types). There is **no transactional outbox** — no
   table written in the same transaction as the mutation from which a relay
   publishes. A crash between commit and enqueue loses the event permanently.
2. **Fan-out**: a listener enqueues `CallWebhookJobsJob`
   (`src/engine/metadata-modules/webhook/jobs/call-webhook-jobs.job.ts`) on
   `MessageQueue.webhookQueue`; it matches subscriptions from the cached
   `flatWebhookMaps` (wildcard grammar `*.created`, `company.*`, `*.*`),
   chunks 20 deliveries per job, and enqueues `CallWebhookJob` **with
   `{ retryLimit: 3 }`**.
3. **Delivery**: `call-webhook.job.ts` POSTs with a 5s timeout, HMAC-SHA256
   signature headers when a secret exists, and SSRF-guarded egress via
   `core-modules/secure-http-client/`. The batch handler runs its chunk with
   `Promise.all`, and `callWebhook()` wraps the POST in a `try/catch` that
   records an audit event (`WEBHOOK_RESPONSE_EVENT`) and **does not rethrow**.

Net semantics: because the catch swallows all delivery failures, BullMQ never
sees a failed job, and `retryLimit: 3` is dead configuration. Combined with
the missing outbox, webhook delivery is **at-most-once, twice over** — events
can be lost before enqueue and deliveries are never retried after failure.

## Alternatives considered (visibly not taken)

- **Transactional outbox + relay** — the standard fix for the emit-after-commit
  gap; absent (no outbox table, no relay/poller in the tree).
- **Rethrow-on-failure to activate retries** — absent; and naive rethrow would
  be wrong anyway: the `Promise.all` chunk means one rethrow re-delivers the
  *whole* chunk, so correct retries require per-delivery jobs or idempotency
  keys (the payload carries no delivery-attempt identifier).
- **Delivery-guarantee documentation** — also absent; consumers cannot learn
  from the API contract that delivery is at-most-once.

## Consequences

**Positive:** the happy path is simple, low-latency, and cheap; no outbox
table growth or relay lag; per-event audit rows (`success`, `status`) exist
for every attempt, making after-the-fact reconciliation *possible*; SSRF
guarding and HMAC signing are genuinely well-executed (the security posture
of each attempt is strong — it's the *persistence* of attempts that's weak).
**Negative:** any consumer-side blip (deploy, 503, timeout) silently drops
events with no retry; worker crashes drop events with no trace at all;
`retryLimit: 3` misleads every future reader into assuming at-least-once;
audit rows record failures but nothing consumes them proactively, so loss is
discovered by the customer, not the operator. The fix is a known, bounded
project: outbox, per-delivery idempotency keys, rethrow, dead-letter alerting.

## Evidence

`src/engine/metadata-modules/webhook/jobs/call-webhook-jobs.job.ts:66-74`
(chunking and the `retryLimit: 3` option);
`call-webhook.job.ts:34-41` (`Promise.all` over the chunk) and `:102-116`
(the swallowing catch); `src/engine/workspace-event-emitter/`;
`core-modules/secure-http-client/`; absence-of-evidence: no outbox entity or
relay job anywhere under `src/engine/`.
