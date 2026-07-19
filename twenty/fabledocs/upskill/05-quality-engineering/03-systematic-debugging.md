# Systematic Debugging

The method: **reproduce → narrow → hypothesize → test cheaply → fix the root cause → add regression coverage.** The narrowing question in this codebase is almost always: *which layer lied?* Front cache / GraphQL / pipeline / ORM-permissions / SQL / worker. Tools per layer: browser devtools + Apollo cache inspector (front), the request's GraphQL payload (network tab), server logs, the read-only Postgres MCP / `psql` for raw data ([CLAUDE.md](../../../CLAUDE.md) "Database Inspection"), BullMQ queue state in Redis, audit events.

---

## Scenario 1: "I created a record and it vanished on refresh"

Reproduction: create a company via UI; it appears; F5; gone.
First question: did the *server* ever have it, or did only the cache?
Narrowing path: 1. Network tab — did `createOneCompany` return 200 with data? If no → the optimistic record masked a failed mutation; check the error and the rollback path ([useCreateOneRecord.ts#L164-L188](../../../packages/twenty-front/src/modules/object-record/hooks/useCreateOneRecord.ts#L164-L188) — did rollback also fail?). 2. If yes → query Postgres for the row. Row exists → the *list query* filter excludes it (check view filters / `shouldMatchRootQueryFilter` in the optimistic effect, [L124-L134](../../../packages/twenty-front/src/modules/object-record/hooks/useCreateOneRecord.ts#L124-L134)). Row absent → soft-deleted? RLS predicate hides it from *your* role?
Useful probes: Apollo cache devtools; `SELECT * FROM <workspace-schema>.company WHERE id = ...` including `"deletedAt"`.
Likely root causes: failed mutation with silent optimistic display; row-level permission predicate excluding the creator (misconfigured role).
Regression test: Recipe 6 (rollback) or an integration test asserting creator-can-read-own-record for the role.
Senior lesson: optimistic UIs convert server errors into "vanishing" bugs — always check the network truth first.
Interview version: narrate the layer-elimination out loud: "First I establish whether the server ever knew about the record — that halves the search space."

## Scenario 2: "Webhook stopped firing for a customer"

Reproduction: customer says `company.updated` webhooks stopped Tuesday.
First question: is the event not emitted, not matched, not enqueued, or not delivered?
Narrowing path: 1. Audit log for `WEBHOOK_RESPONSE_EVENT` — entries with `success: false`? Then delivery is failing (receiver down / SSRF-blocked / timeout — the audit event distinguishes blocked-IP, [call-webhook.job.ts#L103-L114](../../../packages/twenty-server/src/engine/metadata-modules/webhook/jobs/call-webhook.job.ts#L103-L114)). Remember: failures are **not retried** — one 5s attempt. 2. No audit entries at all → check webhook still exists and `operations` patterns match ([call-webhook-jobs.job.ts#L39-L59](../../../packages/twenty-server/src/engine/metadata-modules/webhook/jobs/call-webhook-jobs.job.ts#L39-L59)); check the workspace cache for stale `flatWebhookMaps`. 3. Still nothing → is the worker process running? Is the update actually going through the ORM path that emits (or through a bypass/raw path)?
Useful probes: Redis queue depth for `webhookQueue`; audit table; a test webhook to a request-bin.
Likely root causes: receiver became slow (>5s), silently failing every attempt; webhook operations edited to a non-matching pattern.
Regression test: Recipe 4 variants.
Senior lesson: at-most-once delivery turns "receiver flaked for an hour" into permanent data loss for the consumer — this is where you propose the retry/backoff change with evidence.
Interview version: this is a full "debug a distributed feature" story — producer/router/consumer elimination.

## Scenario 3: "Permission denied, but the role looks right"

Reproduction: user with a custom role gets the standardized permission error on findMany for a custom object.
First question: which of the four authz layers denied ([03-architecture-and-patterns/03](../03-architecture-and-patterns/03-validation-auth-and-permissions.md))?
Narrowing path: 1. Is the object a *system* object requiring a settings flag ([common-base-query-runner.service.ts#L282-L303](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-base-query-runner.service.ts#L282-L303))? 2. Does the role's object permission grant READ (check metadata tables)? 3. Is a *join* in the query touching a forbidden object ([permissions.utils.ts#L279-L360](../../../packages/twenty-server/src/engine/twenty-orm/repository/permissions.utils.ts#L279-L360))? — the error looks identical. 4. Row-level predicates returning zero rows is *not* an error — different symptom, keep straight.
Useful probes: replay the exact GraphQL query minus one relation at a time (bisect the query shape).
Likely root causes: relation to a restricted object inside the requested fields.
Regression test: Recipe 3 with the failing query shape.
Senior lesson: uniform error messages are good security and bad debuggability — the *bisection* technique is the compensating skill.
Interview version: "the error is deliberately opaque; here's how I bisect the query to localize the denial."

## Scenario 4: "Custom field added but doesn't appear in the API"

Reproduction: admin adds field via settings; UI shows it; API client's query errors with "unknown field."
First question: metadata row vs Postgres column vs GraphQL schema cache — which is behind?
Narrowing path: 1. Metadata table has the field row? 2. Workspace schema has the column (check via Postgres)? 3. Does a fresh request get a new schema — is the cached SDL keyed to an *old metadata version* ([workspace-schema.factory.ts#L27-L31](../../../packages/twenty-server/src/engine/api/graphql/workspace-schema.factory.ts#L27-L31); version fetched per request at [jwt-auth.guard.ts#L29-L33](../../../packages/twenty-server/src/engine/guards/jwt-auth.guard.ts#L29-L33))? 4. Client-side: stale codegen types or cached introspection in the API client?
Useful probes: compare metadata version in Redis vs the version the request binds; introspection query for the field.
Likely root causes: cache invalidation missed (version not bumped) or a partially-failed workspace migration (row without column — the desync from [critique risk 4](../03-architecture-and-patterns/06-architecture-critique.md)).
Regression test: integration test for field-create → immediate query.
Senior lesson: in dual-write systems (DDL + metadata + cache), debugging = finding which write lagged; a reconciliation check beats ad-hoc fixes.
Interview version: excellent "cache invalidation war story" material.

## Scenario 5: "Workflow run stuck in RUNNING forever"

Reproduction: run started, no steps progressing, status RUNNING for hours.
First question: did a worker die mid-step, or is a step legitimately waiting?
Narrowing path: 1. Read `workflowRun.state.stepInfos` — which step last wrote output? 2. Queue: is there a pending `RUN_WORKFLOW_JOB` for this run, or was the job consumed and lost with the worker? (Crash after status→RUNNING, before step completion = orphan.) 3. Check the staled-runs safety net: [workflow-handle-staled-runs.command.ts](../../../packages/twenty-server/src/modules/workflow/workflow-runner/workflow-run-queue/command/workflow-handle-staled-runs.command.ts) and [staled-runs-threshold.ts](../../../packages/twenty-server/src/modules/workflow/workflow-runner/workflow-run-queue/constants/staled-runs-threshold.ts) — is it scheduled and running?
Useful probes: worker process uptime/restart logs vs run start time; Redis job state.
Likely root causes: worker crash between persisted status and step enqueue — the gap the status guard can't cover.
Regression test: hard without chaos tooling — instead assert the staled-run sweeper rescues a synthetic stuck run.
Senior lesson: every state machine needs a janitor; the presence of `handle-staled-runs` tells you the team learned this — find the janitor before declaring a system broken.
Interview version: "how do you recover orphaned jobs" — sweeper + threshold + resume-from-step, all real here.

---

Drill: run Scenario 3's bisection on paper for a query with three relations. Self-grade — Strong: your bisection is O(log n) (halve the relations), not O(n) (remove one at a time), and you can say why it matters at 20 relations.
