# 05 — Error handling: taxonomy, filters, and the webhook story I tell every new hire

## What it is and why it exists

Errors here cross four boundaries — GraphQL, REST, background jobs, and the
metadata engine — and each needs different behavior. The design has three tiers:

1. **Typed domain exceptions.** Each module owns its exceptions with error codes:
   `src/engine/twenty-orm/exceptions/twenty-orm.exception.ts`,
   `src/engine/workspace-manager/workspace-migration.exception.ts`,
   `workspace-query-runner.exception.ts`, and so on. The rule: code throws typed
   exceptions with machine-readable codes; *nothing* below the filter layer
   decides what an HTTP status or GraphQL error extension looks like.
2. **Boundary filters/translators.** REST:
   `src/engine/api/rest/rest-api-exception.filter.ts` (31 lines — deliberately
   dumb, it maps codes to statuses). GraphQL: error hooks in the exception-handler
   module. The ORM's `error-handling/` translates raw Postgres errors (constraint
   violations, missing columns) into typed exceptions *before* they escape the
   data layer — a Postgres error message never reaches a client verbatim.
3. **Last-resort catch-all.** `src/filters/unhandled-exception.filter.ts` — 40
   lines whose funniest job is re-adding CORS headers, because an exception thrown
   in middleware *before* the CORS middleware produces a response with no CORS
   headers, and the browser then reports a misleading CORS error instead of the
   real 500. Read the comment at the top of that file; it's a war story in
   two sentences. (Note the `TODO: Check if needed` inside — it is needed; the
   TODO is stale and I should delete it.)

Reporting is centralized in `src/engine/core-modules/exception-handler/` with a
driver interface (`drivers/sentry.driver.ts`, `drivers/console.driver.ts`) so
"where do exceptions go" is configuration, not code.

## The hard decisions

### 1. Exception codes over HTTP-shaped exceptions in domain code

Rejected alternative: NestJS's idiomatic `throw new BadRequestException(...)`
from services. It's seductively convenient and it's wrong for us, because the
same domain error surfaces via GraphQL (no HTTP statuses in-band), REST, MCP,
and job logs. HTTP-shaped exceptions in domain code mean the GraphQL layer
un-translating HTTP to re-translate it. Domain throws domain; edges translate.

The cost: ceremony, again. Every module carries an exception class and code enum,
and lazy paths (`throw new Error(...)`) still exist in older corners — when you
find one, upgrading it is a welcome, always-approved drive-by fix.

### 2. Fail the request vs. fail the operation in the metadata engine

When a metadata migration half-applies (DDL step 3 of 7 fails), what do we do?
Options considered: (a) wrap all DDL in one transaction and roll back, (b) apply
best-effort and report partial success, (c) fail, leave applied steps, and rely
on the flat-diff design to converge on retry.

We do (c), and it falls straight out of doc 01: because desired state is
declarative and the builder diffs against *actual* state, a retry after partial
failure computes only the remaining delta. (a) is impossible in general — some
DDL and data backfills can't share one transaction at our sizes; (b) is how you
get silent schema drift. What bothers me: (c) makes error *reporting* awkward.
The user sees "migration failed, retry," which is honest but reads as flaky even
when the system is behaving exactly as designed. The UX of convergent retry is
an unsolved papercut.

### 3. Swallow vs. rethrow in background jobs — the decision we got wrong once

Queue jobs have a brutal contract: **an unhandled throw means the queue retries;
a caught error means the queue considers the job done.** Every `catch` block in a
job is therefore a *delivery-semantics decision*, not a code-style choice.

Now open `src/engine/metadata-modules/webhook/jobs/call-webhook.job.ts` and hold
it next to `call-webhook-jobs.job.ts`:

- The fan-out job enqueues delivery chunks **with `{ retryLimit: 3 }`** — someone
  clearly intended retries.
- But `CallWebhookJob.callWebhook()` wraps the HTTP POST in a `try/catch` whose
  catch block records an audit event and **does not rethrow**.

Net effect: a customer endpoint returning 500, timing out, or being temporarily
unreachable produces exactly one attempt. The `retryLimit: 3` is dead
configuration — it can only fire on failures *outside* that try/catch, which is
approximately nothing. Webhook delivery is at-most-once, and the two files
disagree about whether that's intended. There's a second, deeper layer: events
are emitted in-process after the database write (via
`src/engine/workspace-event-emitter/`) with no transactional outbox, so a worker
crash between commit and enqueue loses the event entirely. Both are real; both
are on the debt register; the incident below is why.

## Evidence to go read

- `src/filters/unhandled-exception.filter.ts` — the CORS story
- `src/engine/api/rest/errors/` and `rest-api-exception.filter.ts` — code→status mapping
- `src/engine/core-modules/exception-handler/exception-handler.service.ts` and `drivers/`
- `src/engine/twenty-orm/error-handling/` — Postgres error translation
- `src/engine/metadata-modules/webhook/jobs/call-webhook.job.ts:102-116` — the
  catch block discussed above; also note the batch `handle()` uses `Promise.all`,
  so one hypothetical rethrow would retry the *whole chunk*, re-delivering the
  successful ones — fixing this properly requires idempotency keys, not just
  deleting a catch

## A production incident this design caused

The at-most-once webhook design met reality when a large customer's receiving
endpoint deployed and returned 503s for about four minutes. Every record event in
that window generated one delivery attempt, one failure, one audit row — and no
retry, because the catch block ate the failures before BullMQ could see them.
Their downstream system silently missed ~1,900 events; they discovered it via a
reconciliation report *days* later, and so did we, because from our side the
audit log showed failures but nothing paged (see doc 06 for why).

The remediation was a replay script driven by the audit events (`WEBHOOK_RESPONSE_EVENT`
rows with `success: false`). The durable lessons: (1) every `catch` in a job
handler gets interrogated in review — "what delivery semantics does this create?";
(2) retry config that can never fire is worse than no retry config, because it
tells readers a comforting lie. The full fix — outbox + rethrow + per-delivery
idempotency keys — is designed and unbuilt. If you want a meaty first project,
this is the one I'd hand you.

## Before you move on, you should be able to answer:

1. Why must domain code throw coded exceptions instead of NestJS HTTP exceptions,
   given our four API surfaces?
2. Explain precisely why `retryLimit: 3` in `call-webhook-jobs.job.ts` has no
   effect, and why naively rethrowing in `callWebhook()` would create a *new*
   bug (look at `handle()`'s `Promise.all` over the chunk).
3. After a partially-failed metadata migration, why is "just retry" actually
   safe in this system — which doc 01 design property guarantees convergence?
