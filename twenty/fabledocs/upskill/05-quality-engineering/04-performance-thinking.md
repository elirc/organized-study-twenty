# Performance Thinking

Rule zero: **measure before optimizing.** Every claim below is a *where to look*, not a *what is slow* — this clone was never profiled. The skill being taught is hotspot *prediction* followed by verification.

## Performance domains in this repo

| Domain | Likely hotspots | Anchors / existing mitigations |
| --- | --- | --- |
| React render | 10k-cell record tables; per-cell subscriptions | Mitigated by atom families + selector reads ([usePersistField.ts#L187-L189](../../../packages/twenty-front/src/modules/object-record/record-field/ui/hooks/usePersistField.ts#L187-L189)); risk: any cell subscribing to a broad atom |
| Network (front) | aggregate refetch after every create ([useCreateOneRecord.ts#L190-L192](../../../packages/twenty-front/src/modules/object-record/hooks/useCreateOneRecord.ts#L190-L192)); depth-1 default field selection fetching more than the view needs (L58-L64) | Optimistic cache avoids read round-trips |
| Server CPU | per-workspace SDL computation; arg processing on large batches | SDL cached per metadata version ([workspace-schema.factory.ts#L27-L31](../../../packages/twenty-server/src/engine/api/graphql/workspace-schema.factory.ts#L27-L31)) |
| DB | relation fan-out; missing indexes on custom fields; soft-delete filters on big tables | complexity cap + nested-1:N ban ([common-base-query-runner.service.ts#L397-L431](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-base-query-runner.service.ts#L397-L431)); read replicas for read-only ops (L332-L334); index metadata per object |
| Queue/worker | webhook fan-out spikes (mass update → events × webhooks); 5s timeouts serializing worker slots | chunking to 20 ([call-webhook-jobs.job.ts#L66](../../../packages/twenty-server/src/engine/metadata-modules/webhook/jobs/call-webhook-jobs.job.ts#L66)); `Promise.all` within chunk |
| Bundle | Linaria strips runtime CSS cost; watch icon/emoji packs and editor deps (BlockNote) | Vite code-splitting; verify with `npx nx build twenty-front` + analyzer (__inferred__) |
| Memory | flat metadata maps per workspace in cache; worker holding large event batches | version-keyed eviction |

## How to find the classics here

- **N+1:** enable TypeORM query logging locally; load a table view; count queries per row rendered. The generated API's relation loading is centralized — N+1 would come from *feature code* looping repositories (grep `for` loops containing `await` + repository calls in `modules/`).
- **Serial async:** look for sequential awaits on independent I/O (contrast the deliberate serial enqueue at [call-webhook-jobs.job.ts#L68-L74](../../../packages/twenty-server/src/engine/metadata-modules/webhook/jobs/call-webhook-jobs.job.ts#L68-L74) — is it *required* to be serial? A defensible PR question).
- **Expensive renders:** React DevTools profiler on the record table; sort by commit time; any cell rendering on unrelated-cell edits = subscription too broad.
- **Missing indexes:** `EXPLAIN ANALYZE` the SQL from query logging on a workspace with realistic volume; check index metadata for the filtered/sorted fields.
- **Unbounded queries:** any repository call without take/limit in `modules/` — the pipeline caps API queries, but internal jobs can still table-scan (grep `find(` in jobs).

## The senior move: capacity framing

Don't say "webhooks might be slow." Say: "a bulk update of 1,000 records with 5 matching webhooks creates 5,000 deliveries = 250 jobs of 20; at worst-case 5s each with W workers that's 1250/W seconds of queue latency; measure the actual p95 delivery time and decide if that's acceptable." Numbers-first reasoning, even rough, is what distinguishes mid from senior in interviews — this repo gives you the constants to practice with ([WEBHOOK_JOBS_CHUNK_SIZE](../../../packages/twenty-server/src/engine/metadata-modules/webhook/jobs/call-webhook-jobs.job.ts#L19), the 5s timeout, `retryLimit`).

Drill: write the back-of-envelope for "user imports 10k people via spreadsheet import" — mutations, events, webhook jobs, aggregate refetches. Self-grade — Strong: you identify the *front* (refetch per create if imports reuse the single-create path — verify what [spreadsheet-import](../../../packages/twenty-front/src/modules/spreadsheet-import) actually calls; batch hooks exist: [useBatchCreateManyRecords.ts](../../../packages/twenty-front/src/modules/object-record/hooks/useBatchCreateManyRecords.ts)) as a bigger risk than the DB.

## Interview angle

"How would you find/fix a slow page?" → profile → localize domain (table above is your mental checklist) → fix narrowest cause → add a guardrail (complexity cap, index, memo) so it can't regress. Cross-link: [08-interview-prep/02](../08-interview-prep/02-frontend-framework-questions.md) Q8–Q9, [08-interview-prep/03](../08-interview-prep/03-api-and-data-modeling-questions.md) Q10.
