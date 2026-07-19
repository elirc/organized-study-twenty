# Senior Build Projects (5)

Each 2 days–4 weeks. Deliverables are the *plans and slices*, not heroic diffs: problem, value, design checklist, architecture decisions, likely files, migration, tests, security, performance, rollout/rollback, open questions, stretch. Do these in a dev environment; treat upstream acceptance as a stretch goal requiring maintainer discussion first.

---

## Project 1: Transactional Outbox for Record Events (2–4 weeks)

Problem statement: events are emitted in-process after SQL execution ([workspace-insert-query-builder.ts#L255-L275](../../../packages/twenty-server/src/engine/twenty-orm/repository/workspace-insert-query-builder.ts#L255-L275) → [workspace-event-emitter.ts](../../../packages/twenty-server/src/engine/workspace-event-emitter/workspace-event-emitter.ts#L39-L67)); crash/rollback windows can lose or fabricate events feeding webhooks, workflows, and audit.
Product value: "your automation will not silently miss records" — an enterprise trust feature.
Architecture decisions: outbox table location (per-workspace schema vs core with workspaceId — decide against per-workspace DDL cost); relay design (polling with `FOR UPDATE SKIP LOCKED` vs LISTEN/NOTIFY); ordering key (per-record? per-object?); which consumers move to outbox (queue-bound: webhooks/workflows) vs stay in-process (live subscriptions — latency-sensitive).
Likely files: twenty-orm query builders (write outbox row in same transaction), new relay module under `engine/`, message-queue enqueue points.
Migration plan: phase 1 dual-write with reconciliation counter; phase 2 consumers read outbox-sourced queue; phase 3 remove direct enqueue. Feature-flag each phase.
Test plan: kill-the-relay chaos test (zero loss on restart); rollback-transaction test (zero phantom events); ordering test.
Security plan: outbox rows carry record data — same access class as audit.
Performance plan: measure added write latency (one INSERT per write batch); relay throughput budget.
Rollout/rollback: flags per phase; rollback = disable relay, direct enqueue still present through phase 2.
Open questions: retention/cleanup policy; multi-region story.
Stretch: expose delivery guarantees in docs as a product commitment.
Interview story potential: end-to-end exactly-once-ish pipeline design — a staff-level talking point delivered at mid-level depth.

## Project 2: Webhook Reliability Suite (1–2 weeks)

Problem: single-attempt at-most-once delivery with quiet failures ([call-webhook.job.ts#L102-L116](../../../packages/twenty-server/src/engine/metadata-modules/webhook/jobs/call-webhook.job.ts#L102-L116)).
Scope: M1 (retry+backoff per-delivery jobs) + dead-letter after N attempts + per-webhook auto-disable on sustained failure (circuit breaker) + failure metrics (Ticket 7) + settings UI surfacing health (M2).
Key decisions: per-delivery job granularity (fixes the chunk-retry problem); disable thresholds; notification to workspace admins on auto-disable.
Tests: fake receiver matrix (200/429/500/timeout/SSRF-blocked); breaker state transitions.
Rollout: flag per workspace; default off → beta cohort → default on.
Interview story: "designed retries, DLQ, and a circuit breaker around a real constraint set."

## Project 3: Idempotency Keys for the Public API (1–2 weeks)

Problem: API-key clients (Zapier, scripts) retrying createOne/createMany can double-create; client-minted UUIDs help browsers ([useCreateOneRecord.ts#L87](../../../packages/twenty-front/src/modules/object-record/hooks/useCreateOneRecord.ts#L87)) but aren't an API contract.
Design: `Idempotency-Key` header → Redis (or table) keyed `{workspaceId, key}` storing response for TTL; scope to mutating REST endpoints first (interception point: REST handlers → [common runner pipeline](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-base-query-runner.service.ts#L101-L159)).
Decisions: replay returns stored response vs re-executes read; concurrent duplicate (lock or 409); key TTL.
Tests: same-key twice = one record; concurrent race integration test.
Security: keys are per-workspace namespaced — cross-tenant collision impossible by construction.
Interview story: the textbook idempotency question, implemented.

## Project 4: Query Performance Observatory (2–3 weeks)

Problem: metadata-generated queries make per-query performance opaque; slow tenants are found by complaint.
Design: instrument the query runners with duration histograms tagged by object/operation (extend `MetricsService`); slow-query log with parsed filter shapes (no values — PII); admin-panel view.
Decisions: sampling rate; cardinality budget (object × operation only, never per-workspace tags in metrics — put workspace in logs).
Tests: overhead benchmark (<1% target); log redaction test.
Interview story: "built the measurement layer before optimizing — and can quote overhead numbers."

## Project 5: Extension-Safe Public Contract Tests (2–4 weeks)

Problem: the webhook payload, REST shapes, and GraphQL schema are consumed by Zapier, the SDK, and user apps; breaking changes are caught by comments and vigilance ([call-webhook-jobs.job.ts#L34-L37](../../../packages/twenty-server/src/engine/metadata-modules/webhook/jobs/call-webhook-jobs.job.ts#L34-L37)).
Design: snapshot-based contract test suite: generated SDL for a seeded workspace, REST response schemas, webhook payload fixtures — diffed in CI with an explicit "breaking-change acknowledged" mechanism (the repo already has a [ci-breaking-changes.yaml](../../../.github/workflows/ci-breaking-changes.yaml) workflow — study it first; your project may be "extend it to webhooks/REST" — verify current coverage).
Tests: the suite *is* tests; meta-test: introduce a deliberate break, assert CI fails.
Interview story: "consumer-driven contracts in CI — how I made breaking changes a decision instead of an accident."
