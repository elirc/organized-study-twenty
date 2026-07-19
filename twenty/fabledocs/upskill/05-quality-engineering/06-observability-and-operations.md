# Observability & Operations

Observability = can you answer "what did the system do and why" without adding code. Inventory of what exists here, then the flow-by-flow "how would I know it broke?"

## The instruments

| Instrument | Where | Notes |
| --- | --- | --- |
| Structured logging | `LoggerService` wired at boot ([main.ts#L46-L62](../../../packages/twenty-server/src/main.ts#L46-L62)); per-class `Logger` (e.g. [run-workflow.job.ts#L27](../../../packages/twenty-server/src/modules/workflow/workflow-runner/jobs/run-workflow.job.ts#L27)) | Buffered logs option via env |
| Error tracking | `./instrument` imported first in [main.ts#L21](../../../packages/twenty-server/src/main.ts#L21) ([instrument.ts](../../../packages/twenty-server/src/instrument.ts) — Sentry-style init; open it) + global [unhandled-exception.filter.ts](../../../packages/twenty-server/src/filters/unhandled-exception.filter.ts) | Import order matters for instrumentation — say why in interviews (it must patch modules before they load) |
| Metrics | `MetricsService` counters keyed by `MetricsKeys` — e.g. rate-limit rejections ([common-base-query-runner.service.ts#L387-L394](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-base-query-runner.service.ts#L387-L394)), webhook completions, workflow trigger counts ([run-workflow.job.ts#L213-L243](../../../packages/twenty-server/src/modules/workflow/workflow-runner/jobs/run-workflow.job.ts#L213-L243)) | Fire-and-forget by design |
| Audit events | `AuditService.insertWorkspaceEvent` — per-tenant behavioral record, e.g. webhook attempts success *and* failure ([call-webhook.job.ts#L92-L115](../../../packages/twenty-server/src/engine/metadata-modules/webhook/jobs/call-webhook.job.ts#L92-L115)) | The tenant-visible layer of observability |
| Health | [engine/core-modules/health](../../../packages/twenty-server/src/engine/core-modules/health) | Liveness/readiness for orchestration |
| Admin panel | [engine/core-modules/admin-panel](../../../packages/twenty-server/src/engine/core-modules/admin-panel) + impersonation (guarded) | Operator toolbox |
| Front error boundaries | error-handler module, [modules/error-handler](../../../packages/twenty-front/src/modules/error-handler) | Contain render crashes |

Deploy/rollback: Docker images via [twenty-docker](../../../packages/twenty-docker); CD workflows [cd-deploy-main.yaml](../../../.github/workflows/cd-deploy-main.yaml) / tag-based releases; DB rollback story = instance-command `down` logic ([CLAUDE.md](../../../CLAUDE.md)). Rollback of *code* is cheap; rollback of *workspace DDL* is the expensive part — that asymmetry drives the up/down discipline.

## "How would I know this broke?" per key flow

| Flow | First signal | Gap to know about |
| --- | --- | --- |
| Record CRUD | API error rates (Sentry/filter), front error boundary reports | Optimistic UI can mask intermittent failures from *users* — error tracking is your only honest witness |
| Webhooks | Audit events with `success: false`; `JobWebhookCallCompleted` counter flatlining | No retries → no retry-exhausted alarm exists; a receiver outage looks like *quiet audit rows*, not an alert. If you own this: alert on failure-rate over window |
| Workflows | Run statuses (FAILED visible in product UI); staled-runs sweeper | Sweeper is the safety net — monitor that the *janitor itself* runs |
| Metadata changes | Schema errors surface as empty schema fallback ([workspace-schema.factory.ts#L33-L35](../../../packages/twenty-server/src/engine/api/graphql/workspace-schema.factory.ts#L33-L35)) | Empty-schema fallback trades a loud 500 for a quiet "everything's gone" — know this when triaging "my API is empty" reports |
| Queues | Redis depth, BullMQ failed-job counts | Worker down = queues grow silently; depth alerting is an ops must |
| Email/calendar sync | monitoring dir in messaging module ([modules/messaging/monitoring](../../../packages/twenty-server/src/modules/messaging/monitoring)) | Third-party API quotas fail in bursts |

## The transferable habit

For every feature you ship, write one sentence in the PR: *"If this breaks, we'll know because ___."* If the blank is "a user tells us," add the instrument. This repo's webhook audit trail is a model: failure is recorded with the same fidelity as success ([call-webhook.job.ts#L108-L115](../../../packages/twenty-server/src/engine/metadata-modules/webhook/jobs/call-webhook.job.ts#L108-L115)) — the *absence of retries* is compensated by *presence of evidence*. Whether that's enough is a product decision; that it was made consciously is the senior signal.

Drill: pick any junior ticket from [06-contribution-practice/01-good-first-tickets.md](../06-contribution-practice/01-good-first-tickets.md) and write its "we'll know because" sentence. Strong: your signal is queryable (metric/audit), not anecdotal.

Interview angle: "How do you monitor a background job system?" — queue depth, failure counts, staleness sweepers, and per-attempt audit; all four exist here to point at. ([08-interview-prep/04](../08-interview-prep/04-system-design-from-this-repo.md), scaling step)
