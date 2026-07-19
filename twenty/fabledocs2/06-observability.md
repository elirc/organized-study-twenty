# 06 — Observability: logger, metrics, audit, Sentry — and the void operator problem

## What it is and why it exists

Observability here is four systems with four different customers:

1. **Logger** — `src/engine/core-modules/logger/` (driver-based behind
   `logger.interface.ts`, configured by `logger.module-factory.ts`). For engineers
   reading a stream. Standard NestJS `Logger` semantics throughout the codebase.
2. **Exception reporting** — `src/engine/core-modules/exception-handler/` with
   `drivers/sentry.driver.ts` (Sentry init lives early in boot:
   `src/instrument.ts`, plus `core-modules/sentry/`). For engineers being paged.
3. **Metrics** — `src/engine/core-modules/metrics/metrics.service.ts` and
   `metrics-cache.service.ts`. Counters keyed by `MetricsKeys`, some cached in
   Redis for cheap aggregation, surfaced in the admin panel and health
   endpoints (`core-modules/health/`). For dashboards and capacity questions.
4. **Audit / workspace events** — `src/engine/core-modules/audit/` (read its
   `README.md`). `AuditService.createContext({workspaceId})` returns typed event
   emitters (`insertWorkspaceEvent`, `createObjectEvent`, `createPageviewEvent`).
   This one is different in kind: it's **a product feature and a forensic record**,
   not just telemetry — customer-visible timelines, usage analytics, and (see
   docs 03 and 05) the raw material for incident replay scripts. Analytics-scale
   event storage goes to ClickHouse when enabled; there's also
   `core-modules/event-logs/` and `application-logs/` for adjacent trails.

Know which of the four you're feeding before you instrument anything. The most
common review comment I write on instrumentation PRs is "this is an audit event,
not a log line" or vice versa.

## The hard decisions

### 1. Audit events as first-class typed constants vs. free-form tracking calls

Every audit event is a typed constant with a schema — e.g.
`WEBHOOK_RESPONSE_EVENT` in
`src/engine/core-modules/audit/utils/events/workspace-event/webhook/webhook-response.ts`,
with the full catalog under `audit/utils/events/`. The rejected alternative was
the industry default: `track('webhook_response', {...whatever})` with stringly
names and ad-hoc payloads.

Why rejected: free-form tracking rots into a swamp within a year — misspelled
event names, three shapes for the same event, dashboards querying fields that
stopped being sent. Typed constants mean the compiler knows every event we can
emit and its shape; when the incident-replay script in doc 05 needed `success`
and `status` on every webhook response event, we *knew* they'd be there.

The tradeoff: friction. Adding an event means adding a file, and people route
around friction — features ship with no audit trail because "we'll instrument
later." I accept this cost deliberately: **missing events are visible and fixable;
corrupt event data is forever.** But it means part of *your* job in review is
asking "where's the audit event?" on any user-facing mutation.

### 2. Fire-and-forget observability writes — the `void` operator

Look at any call site: `void auditService.insertWorkspaceEvent(...)`,
`void this.metricsService.incrementCounter(...)` (both visible in
`call-webhook.job.ts`). The `void` is a deliberate statement: observability
writes must never block, fail, or slow down the operation they observe. Rejected
alternative: awaiting them (adds a Redis/ClickHouse round-trip to every hot-path
operation and, worse, couples product availability to telemetry availability).

The tradeoff that genuinely bothers me — this is the one I'd flag in an
architecture review of my own system: **fire-and-forget means silently lossy.**
If the audit insert rejects, nothing notices; there's no local buffer, no dead
letter, not even a counter of dropped events (a metric about failing metrics has
an obvious chicken-and-egg smell, but a best-effort one is still better than
blindness). Our forensic record — the thing we used to *recover from* two
incidents in these docs — is itself delivered at-most-once. The honest status:
acceptable for analytics, uncomfortable for audit, and the eventual fix is the
same transactional-outbox work as webhook delivery (doc 05), because it's
literally the same problem shape.

### 3. Push counters into Redis vs. run real time-series infrastructure

`MetricsService` with `metrics-cache.service.ts` implements counters over the
cache layer rather than wiring Prometheus/OpenTelemetry metrics through
everything. Chosen because this codebase is **self-hosted by strangers**: every
observability dependency we hard-require becomes a barrier for someone running
Twenty on a $10 VPS. Redis is already required; a metrics stack isn't. Same logic
drove the driver patterns everywhere — logger drivers, exception drivers
(console vs Sentry), queue drivers: cloud-us gets the full stack, self-hosters
get working software with defaults.

What bothers me: our own cloud operation outgrew this. Counters in Redis answer
"how many," never "how slow" — no histograms, no percentiles, no exemplars. We
bolt on Sentry performance tracing for latency questions, which means two tools
with two sampling stories that don't join. If you're profiling a slow endpoint,
the truth is in Sentry traces, not in `MetricsService`, and knowing that will
save you an hour.

## Evidence to go read

- `src/engine/core-modules/audit/README.md` and `audit/utils/events/` — the event catalog
- `src/engine/core-modules/metrics/metrics.service.ts` — note `shouldStoreInCache`
- `src/instrument.ts` — Sentry bootstrapping, and why it's imported before the app
- `src/engine/core-modules/exception-handler/drivers/` — the console/Sentry split
- `src/engine/core-modules/health/` — what "healthy" formally means for this system
- `grep -r "void this.auditService" packages/twenty-server/src | wc -l` — feel the
  scale of the fire-and-forget surface yourself

## A production incident this design shaped (both directions)

Same incident as doc 05 — the ~1,900 lost webhook events — seen through the
observability lens, because it's a perfect specimen of this subsystem's strength
and weakness in one story.

**What worked:** every failed delivery had written a typed `WEBHOOK_RESPONSE_EVENT`
with `success: false` and the target URL. The replay script was a query plus a
loop; recovery took hours instead of weeks. Decision #1 (typed events) paid for
itself in that one afternoon.

**What failed:** nothing *told* us. Failures were recorded, not surfaced — we had
an audit trail and no alarm wired to it. Four minutes of 503s produced 1,900
perfectly-formatted rows that nobody looked at until the customer called. The
post-incident fix was threading webhook failure counts into `MetricsService`
(you can see `MetricsKeys.JobWebhookCallCompleted` incremented in
`call-webhook.job.ts` — note it counts *completions*; the failure-rate alerting
derived from the gap is configured on the ops side) and an admin-panel surface
for webhook health. The generalizable lesson, which I'll repeat until you're
tired of it: **recording is not observing. If no human or alert consumes a
signal, you don't have observability — you have a diary.**

## Before you move on, you should be able to answer:

1. An event needs to be tracked: how do you decide among logger, metrics, audit
   events, and Sentry — who is the consumer of each?
2. Why are audit/metrics writes `void`-ed, and what property of the audit trail
   does that sacrifice? Why is that sacrifice more acceptable for pageview
   analytics than for webhook delivery records?
3. During the webhook incident, the data needed for recovery already existed.
   What was missing, and what's the difference between "recorded" and "observed"
   in concrete, wire-something-up terms?
