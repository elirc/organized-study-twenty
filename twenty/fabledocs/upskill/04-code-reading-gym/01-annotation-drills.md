# Annotation Drills

For each excerpt (open the real file at the anchor — excerpts here are truncated on purpose), annotate in writing: **inputs** (and who controls them), **outputs**, **dependencies**, **invariants** (what must always hold), **side effects**, **failure modes**. Then check against the notes and grade with the rubric at the bottom.

---

## Drill 1 — HMAC signature
[call-webhook.job.ts#L23-L32](../../../packages/twenty-server/src/engine/metadata-modules/webhook/jobs/call-webhook.job.ts#L23-L32) (`generateSignature`)
Notes to check yourself against: input `payload` is attacker-influenceable (record data), `secret` is tenant config; invariant: signature must cover the timestamp (it does — `${timestamp}:${JSON.stringify(payload)}`), or replay defense dies; failure mode: `JSON.stringify` key order — if the receiver re-serializes, verification breaks; the *string received on the wire* must be what's verified.

## Drill 2 — Fan-out matching
[call-webhook-jobs.job.ts#L39-L59](../../../packages/twenty-server/src/engine/metadata-modules/webhook/jobs/call-webhook-jobs.job.ts#L39-L59)
Check: input event name is split on `.` exactly once into two parts — an event with no dot yields `operation === undefined` and match strings like `undefined.*` (can that happen? find who computes event names: [workspace-event-emitter.ts#L61](../../../packages/twenty-server/src/engine/workspace-event-emitter/workspace-event-emitter.ts#L61)); side effect: enqueues jobs; invariant: cache must reflect current webhooks (dependency on invalidation).

## Drill 3 — Throttling
[common-base-query-runner.service.ts#L350-L395](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-base-query-runner.service.ts#L350-L395)
Check: early return for non-API-key contexts (browsers not throttled *here*); two buckets (burst + sustained) keyed per workspace — meaning one workspace's keys share a budget (all API keys of a workspace share the bucket: is the key per-workspace or per-key? read the key template at L357 — it's per-workspace); failure mode: metric increment happens in `catch` then rethrows — the metric fires only on rejection.

## Drill 4 — Optimistic rollback
[useCreateOneRecord.ts#L164-L188](../../../packages/twenty-front/src/modules/object-record/hooks/useCreateOneRecord.ts#L164-L188)
Check: rollback happens only if an optimistic record was actually cached (`if (!isDefined(recordCreatedInCache)) throw`); two cleanup calls (cache delete + destroy effect) because the record was inserted in *two* representations; the error still rethrows — rollback isn't swallowing.

## Drill 5 — Settings-permission gate
[common-base-query-runner.service.ts#L274-L311](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-base-query-runner.service.ts#L274-L311)
Check: gate applies only to objects listed in a constant map (`OBJECTS_WITH_SETTINGS_PERMISSIONS_REQUIREMENTS`) — adding a new sensitive system object requires *remembering to update that map* (a convention-based invariant, weaker than structural); note both userWorkspaceId and apiKeyId are passed — API keys have permissions too.

## Drill 6 — Workflow resume routing
[run-workflow.job.ts#L134-L211](../../../packages/twenty-server/src/modules/workflow/workflow-runner/jobs/run-workflow.job.ts#L134-L211)
Check: inputs come from the *queue message* (`lastExecutedStepId`) and the *DB state* (`workflowRun.state`) — two sources that can disagree; the code trusts DB state for steps but the message for which step ran; failure mode: step removed from flow between executions → exception path L157-L162; invariant: run must be RUNNING to resume.

## Drill 7 — Schema factory
[workspace-schema.factory.ts#L23-L68](../../../packages/twenty-server/src/engine/api/graphql/workspace-schema.factory.ts#L23-L68)
Check: empty-schema fallback when SDL missing (L33-L35) — availability over correctness: a workspace with broken metadata gets an empty API instead of 500s; resolvers are rebuilt per call while SDL is cached — is resolver creation cheap? (closures over context — yes, relatively); dependency: `buildObjectIdByNameMaps` couples resolver wiring to name uniqueness (invariant: singular names unique per workspace).

## Drill 8 — Field persist dispatch
[usePersistField.ts#L184-L281](../../../packages/twenty-front/src/modules/object-record/record-field/ui/hooks/usePersistField.ts#L184-L281)
Check: three different persist shapes (FK write for relations, morph multi-key write, plain field write); equality checks prevent no-op mutations (L191-L194, L266-L268) — client-side write coalescing; side effects: mutation + Jotai store write (L278-L281); failure mode: the final `else` throws for unimplemented types — runtime, not compile-time, exhaustiveness.

---

## Self-grading rubric (apply per drill)

- **Basic:** correctly listed inputs/outputs and one side effect.
- **Solid:** named the invariant in your own words and who controls each input (user / tenant admin / system).
- **Strong:** identified a failure mode *not stated in the code comments*, and said what test would pin it.

Score 8 drills; anything below Solid, redo in a week. Strong across 6+ means you're reading at mid-level.
