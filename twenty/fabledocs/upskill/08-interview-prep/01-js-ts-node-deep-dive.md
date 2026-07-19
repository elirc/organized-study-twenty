# JS / TS / Node Deep-Dive — 14 Question Cards

Practice protocol: answer aloud in ≤90 seconds *before* reading the card body. Every card: what it tests, repo anchor, junior/mid/senior answer contrast, follow-ups, drill.

---

## Q1: Explain the event loop. Where do promises fit?
Round: JS deep-dive. Testing: mental model of async execution.
Repo anchor: `Promise.all` webhook fan-out — [call-webhook.job.ts#L36-L41](../../../packages/twenty-server/src/engine/metadata-modules/webhook/jobs/call-webhook.job.ts#L36-L41)
Junior: "JS is single-threaded with a callback queue."
Mid adds: microtask vs macrotask ordering; `await` yields; I/O concurrency without threads — "20 webhook POSTs interleave on one thread because they're network-bound."
Senior includes: CPU-bound work (schema SDL computation) starves the loop — hence caching/offloading; when a `Promise.all` of 20 five-second timeouts pins a worker slot, concurrency ≠ throughput.
Follow-ups: microtask starvation; `setImmediate` vs `setTimeout(0)`.
Drill: whiteboard the timeline of the 20-POST `Promise.all` with one slow receiver.

## Q2: Promise.all vs allSettled vs sequential await — when each?
Round: JS. Testing: failure-mode reasoning, not API recall.
Repo anchor: parallel delivery ([call-webhook.job.ts#L36-L41](../../../packages/twenty-server/src/engine/metadata-modules/webhook/jobs/call-webhook.job.ts#L36-L41)) vs serial enqueue ([call-webhook-jobs.job.ts#L68-L74](../../../packages/twenty-server/src/engine/metadata-modules/webhook/jobs/call-webhook-jobs.job.ts#L68-L74))
Junior: definitions.
Mid adds: `all` fails fast — but note Twenty's inner calls each catch their own errors, so `all` there never rejects (an `allSettled` in disguise); serial when ordering/backpressure matters.
Senior includes: unbounded parallelism as a resource bug; chunking as the compromise (chunk of 20).
Drill: explain why `handle()` can use `Promise.all` safely *only because* `callWebhook` swallows.

## Q3: How do you handle errors in async code?
Round: JS. Testing: whether you have a *policy*, not just try/catch syntax.
Repo anchor: swallow-with-audit ([call-webhook.job.ts#L102-L116](../../../packages/twenty-server/src/engine/metadata-modules/webhook/jobs/call-webhook.job.ts#L102-L116)) vs record-and-rethrow ([run-workflow.job.ts#L63-L73](../../../packages/twenty-server/src/modules/workflow/workflow-runner/jobs/run-workflow.job.ts#L63-L73))
Junior: "wrap in try/catch."
Mid adds: the boundary decision — is the error data (log, continue) or control flow (propagate to retry machinery)? Two real examples, opposite policies, both defensible.
Senior includes: typed exceptions with codes as API contract; global unhandled-rejection safety net ([main.ts#L64](../../../packages/twenty-server/src/main.ts#L64)).
Follow-up: what happens to an unhandled rejection in Node 24? (Process crash — hence the filter.)

## Q4: What is a closure? Give a non-toy use.
Round: JS. Repo anchor: resolver factories capturing schema context — [create-one-resolver.factory.ts#L28-L64](../../../packages/twenty-server/src/engine/api/graphql/workspace-resolver-builder/factories/create-one-resolver.factory.ts#L28-L64)
Junior: "a function remembering its scope."
Mid adds: factory-built request handlers closing over build-time context; React `useCallback` closures and stale-capture bugs ([usePersistField.ts#L292-L299](../../../packages/twenty-front/src/modules/object-record/record-field/ui/hooks/usePersistField.ts#L292-L299)).
Senior includes: closures as capability handles (the closure *is* the authorized context); memory implications of capturing large maps.

## Q5: When is swallowing an error acceptable?
Round: JS/judgment. Repo anchor: `void auditService.insert...` fire-and-forget ([call-webhook.job.ts#L92-L101](../../../packages/twenty-server/src/engine/metadata-modules/webhook/jobs/call-webhook.job.ts#L92-L101))
Junior: "never."
Mid adds: telemetry-grade effects may fail silently; business effects never; the `void` keyword as documented intent.
Senior includes: the webhook case — swallowing *delivery* failures is a product decision about retry semantics, and it should be tested so changing it is deliberate (Ticket 1).

## Q6: How does per-request context work in Node without passing parameters everywhere?
Round: Node. Repo anchor: `executeInWorkspaceContext` + `getWorkspaceContext()` ([common-base-query-runner.service.ts#L144-L153, L316](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-base-query-runner.service.ts#L144-L153))
Junior: "middleware sets req.user."
Mid adds: AsyncLocalStorage — context follows the async chain; Twenty scopes tenant/auth this way below the request layer.
Senior includes: loss across unwrapped boundaries (emitters, queues) — which is why queue payloads carry `workspaceId` explicitly; testing implications (context must be entered in tests too).

## Q7: Node vs browser: what breaks when code moves between them?
Round: Node. Repo anchor: `twenty-shared` runtime-neutrality; `node:crypto` HMAC ([call-webhook.job.ts#L1, L28-L32](../../../packages/twenty-server/src/engine/metadata-modules/webhook/jobs/call-webhook.job.ts#L28-L32))
Mid adds: crypto/fs/net vs WebCrypto/fetch; a shared package as the enforced neutral zone.
Senior includes: the *worker* as a third runtime — same code, no HTTP request context; `@Processor` classes must never assume a user.

## Q8: unknown vs any?
Round: TS. Repo anchor: `valueToPersist: unknown` + guards ([usePersistField.ts#L85-L182](../../../packages/twenty-front/src/modules/object-record/record-field/ui/hooks/usePersistField.ts#L85-L182)); repo bans `any`.
Junior: "any disables checking."
Mid adds: `unknown` forces narrowing at boundaries — wire data, user input; show the guard-pair idiom.
Senior includes: exhaustiveness gap of guard chains vs discriminated-union `switch` + `never` (the runtime throw at L282-L290 as evidence).

## Q9: How does narrowing work? Discriminated unions?
Round: TS. Repo anchor: auth-context narrowing via `isUserAuthContext`/`isApiKeyAuthContext` ([common-base-query-runner.service.ts#L292-L302](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-base-query-runner.service.ts#L292-L302))
Mid adds: custom type predicates (`x is T`); narrowing as the authorization branch for three principal types.
Senior includes: predicate lies (a wrong guard is a hole in the proof); keeping predicates colocated with the types they guard.

## Q10: Generics — a real use beyond arrays?
Round: TS. Repo anchor: `CommonBaseQueryRunnerService<Args, Output>` ([common-base-query-runner.service.ts#L60-L63](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-base-query-runner.service.ts#L60-L63))
Mid adds: a generic base pipeline: shared logic once, per-operation precision preserved; constraints (`extends CommonQueryArgs`) as the contract.
Senior includes: where generics stop helping — the runtime-defined schema means records are stringly-typed maps past a point; know the boundary between static and dynamic typing in your design.

## Q11: What is dependency injection and why bother?
Round: Node/architecture. Repo anchor: the runner's injected collaborators ([common-base-query-runner.service.ts#L64-L95](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-base-query-runner.service.ts#L64-L95)); Ticket 1's mock-based test as the payoff.
Mid adds: testability (swap `SecureHttpClientService` with a fake), lifecycle control (request scope, [run-workflow.job.ts#L25](../../../packages/twenty-server/src/modules/workflow/workflow-runner/jobs/run-workflow.job.ts#L25)).
Senior includes: DI containers hide construction cost and scope bugs (request-scoped into singleton); property vs constructor injection tradeoffs in inheritance hierarchies.

## Q12: CJS vs ESM — what actually breaks?
Round: Node. Repo anchor: `graphql-upload/graphqlUploadExpress.mjs` explicit import ([main.ts#L9](../../../packages/twenty-server/src/main.ts#L9))
Mid adds: default-export interop, `require` of ESM-only packages failing, dual-package hazard; the `.mjs` import as a visible seam.
Senior includes: monorepo dimension — one package's module format constrains consumers; resolutions pinning graphql to one instance ([package.json#L28](../../../package.json#L28)) to avoid the two-realms bug.

## Q13: Monorepo vs polyrepo?
Round: tooling. Repo anchor: workspace list + Nx ([package.json#L46-L60](../../../package.json#L46-L60))
Mid adds: atomic cross-layer changes vs build orchestration cost; task graph + caching + affected-only CI as the mitigations.
Senior includes: ownership/contract discipline still required inside a monorepo — shared package as the only sanctioned coupling point.

## Q14: How would you rate-limit an API in Node?
Round: Node/system. Repo anchor: token buckets, short+long windows, per-workspace keys, API-key-only scope ([common-base-query-runner.service.ts#L350-L395](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-base-query-runner.service.ts#L350-L395))
Junior: "use express-rate-limit."
Mid adds: token bucket vs fixed window (bursts); state in Redis for multi-instance; two windows for burst + sustained abuse.
Senior includes: choosing the key (per key? per workspace? per IP?) is the product decision; 429 contract + visibility headers (Ticket M4); what's deliberately *not* limited (user tokens here) and what compensates.

---

Scoring your practice: for each card, did your answer include a repo example unprompted? That habit — not more facts — is the two-week win.
