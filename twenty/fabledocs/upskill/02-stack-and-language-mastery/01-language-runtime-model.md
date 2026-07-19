# Language & Runtime Model (JS/Node)

## The event loop, seen through this repo

Mental model: one call stack; macrotask queue (timers, I/O callbacks) and microtask queue (resolved promises) drained between tasks; `async/await` is syntax over promises. Nothing runs "in parallel" in one Node process — concurrency is interleaving at `await` points.

Where Twenty leans on this:

- **Serial vs parallel awaits.** [call-webhook.job.ts#L36-L41](../../../packages/twenty-server/src/engine/metadata-modules/webhook/jobs/call-webhook.job.ts#L36-L41) uses `Promise.all` to fire up to 20 webhook POSTs concurrently — I/O-bound work interleaves well. Contrast [call-webhook-jobs.job.ts#L68-L74](../../../packages/twenty-server/src/engine/metadata-modules/webhook/jobs/call-webhook-jobs.job.ts#L68-L74), which enqueues chunks with `await` in a `for...of` loop — serial, deliberate, keeps queue pressure orderly. Being able to say *when* you'd choose each is a classic screen question.
- **Fire-and-forget with `void`.** [call-webhook.job.ts#L92-L101](../../../packages/twenty-server/src/engine/metadata-modules/webhook/jobs/call-webhook.job.ts#L92-L101) writes audit events and metrics with `void auditService.insertWorkspaceEvent(...)` — intentionally unawaited so telemetry can't slow or fail the main path. Failure mode: those promises can reject silently; that's acceptable for metrics, catastrophic for business writes. Knowing the difference is the transferable judgment.
- **Request context without passing parameters.** The pipeline reads workspace context via `getWorkspaceContext()` ([common-base-query-runner.service.ts#L316](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-base-query-runner.service.ts#L316)) after entering `executeInWorkspaceContext` ([L144-L153](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-base-query-runner.service.ts#L144-L153)). That's AsyncLocalStorage-style ambient context: state follows the async call chain, not the stack. Failure mode: context loss across un-instrumented boundaries (raw callbacks, event emitters) — exactly why the emitter-to-queue hop in Flow 4 carries `workspaceId` explicitly in the payload instead.

## Error propagation across async boundaries

Two deliberate, opposite policies in the same codebase:

- Webhook delivery **swallows** errors ([call-webhook.job.ts#L102-L116](../../../packages/twenty-server/src/engine/metadata-modules/webhook/jobs/call-webhook.job.ts#L102-L116)): the job never fails, the queue never retries, failure is recorded in audit.
- Workflow execution **rethrows** after marking the run failed ([run-workflow.job.ts#L63-L73](../../../packages/twenty-server/src/modules/workflow/workflow-runner/jobs/run-workflow.job.ts#L63-L73)): the queue sees a failure and can retry/alert.

Transferable rule: *decide at every async boundary whether an error is data (record it, continue) or control flow (propagate it)*. Unhandled promise rejections in Node ≥15 crash the process; the global safety net is [unhandled-exception.filter.ts](../../../packages/twenty-server/src/filters/unhandled-exception.filter.ts) registered in [main.ts#L64](../../../packages/twenty-server/src/main.ts#L64).

## Closures doing real work

Generated resolvers are closures: [create-one-resolver.factory.ts#L28-L64](../../../packages/twenty-server/src/engine/api/graphql/workspace-resolver-builder/factories/create-one-resolver.factory.ts#L28-L64) captures `internalContext` at schema-build time and reuses it per request. Same trick on the front: `persistField` is a `useCallback` closure over metadata and store handles ([usePersistField.ts#L77-L299](../../../packages/twenty-front/src/modules/object-record/record-field/ui/hooks/usePersistField.ts#L77-L299)). Failure mode both places: **stale capture** — if captured data can change (metadata version bump), the closure must be rebuilt; that's why schemas are cached per metadata version and hooks declare dependency arrays.

## Node vs browser vs worker constraints

- `twenty-shared` runs in all three → no Node builtins, no DOM (see [04-tooling-and-build-system.md](04-tooling-and-build-system.md)).
- Crypto: server uses `node:crypto` HMAC ([call-webhook.job.ts#L28-L32](../../../packages/twenty-server/src/engine/metadata-modules/webhook/jobs/call-webhook.job.ts#L28-L32)); the browser would need WebCrypto — an easy porting trap.
- CPU-bound work (schema SDL computation) blocks the event loop; that's one reason it's cached aggressively ([workspace-schema.factory.ts#L27-L31](../../../packages/twenty-server/src/engine/api/graphql/workspace-schema.factory.ts#L27-L31)).

## Pitfall checklist

- [ ] `await` in a loop when order doesn't matter → serialize latency; but unbounded `Promise.all` → resource spikes (Twenty chunks to 20 first).
- [ ] `void promise` on anything a user depends on.
- [ ] Throwing non-`Error` values — loses stack traces across async hops.
- [ ] Assuming two `await`-separated statements are atomic — anything can interleave between them (see the workflow status-check race note in Flow 5).

## Drills

1. In [call-webhook.job.ts#L36-L41](../../../packages/twenty-server/src/engine/metadata-modules/webhook/jobs/call-webhook.job.ts#L36-L41), one receiver takes 5s (timeout) and 19 take 50ms. How long does the job take, and what would change with a `for...of`+`await`? Self-grade — Strong: ~5s vs ~6s, plus you mention head-of-line blocking.
2. Find every `void ` prefix in `call-webhook.job.ts` and classify each as safe/unsafe fire-and-forget. Strong: you articulate the rule you used.

## Interview angle

- "Explain the event loop / microtasks vs macrotasks" → answer, then anchor: "Twenty fires 20 webhook calls with Promise.all because they're I/O-bound…" ([08-interview-prep/01](../08-interview-prep/01-js-ts-node-deep-dive.md), Q1–Q3)
- "Promise.all vs allSettled vs sequential await" → Q4
- "How does context propagate through async code without parameter drilling?" → Q6
- "When is swallowing an error correct?" → Q5
