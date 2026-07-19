# Trace Tables

Fill in each table yourself first (columns: step, file/line, value shape, owner, transformation, risk), then compare. Partial keys are given; the discipline is writing the **value shape** at every hop — that's where bugs hide.

---

## Trace 1 (UI → API): user types "Acme" into a company name cell and presses Enter

| Step | File | Value shape | Owner | Transformation | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | field input component (record-field/ui/meta-types/input) | `"Acme"` raw string | browser | keystrokes → draft state | — |
| 2 | [usePersistField.ts#L97-L98](../../../packages/twenty-front/src/modules/object-record/record-field/ui/hooks/usePersistField.ts#L97-L98) | `unknown` → narrowed text value | browser | guard pair validates def+value | wrong guard = runtime throw (L282-L290) |
| 3 | [usePersistField.ts#L266-L268](../../../packages/twenty-front/src/modules/object-record/record-field/ui/hooks/usePersistField.ts#L266-L268) | old vs new comparison | browser | deep-equal no-op suppression | stale `currentValue` from store |
| 4 | [usePersistField.ts#L270-L281](../../../packages/twenty-front/src/modules/object-record/record-field/ui/hooks/usePersistField.ts#L270-L281) | `{ name: "Acme" }` update input | browser | field name → input object; store updated optimistically | mutation fails after store write → UI/DB divergence until cache reconciles |
| 5 | useUpdateOneRecord → Apollo | GraphQL mutation variables | wire | typed op from codegen | — |
| 6 | update-one resolver factory → runner pipeline | `UpdateOneResolverArgs` | server | graphqlFields(info) → selectedFields | — |
| 7 | update query builder | SQL `UPDATE ... SET name=$1` | orm | permission validation + RLS predicates first | — |
| 8 | [workspace-update-query-builder.ts](../../../packages/twenty-server/src/engine/twenty-orm/repository/workspace-update-query-builder.ts) events | `DatabaseBatchEvent` UPDATED | orm | before/after diff payload | consumers assume diff correctness |

Your task: verify steps 6–8 by opening the files (find the update resolver factory and the update query builder's emit call yourself). Risk column for step 4 is the interview-worthy row.

## Trace 2 (persistence): what exactly hits Postgres for `createOneCompany`

Start: [workspace-insert-query-builder.ts#L160-L275](../../../packages/twenty-server/src/engine/twenty-orm/repository/workspace-insert-query-builder.ts#L180-L275). Rows to produce: values-set preprocessing (files fields L182-L198, nested relations L200-L220), RLS validation (L222), INSERT (L224), the *second* SELECT for event payloads (L229-L246) — note it runs with `shouldBypassPermissionChecks: true` and `noFormatting`, then formats explicitly (L248-L253). Question to answer in your table: how many SQL statements total, and which could fail independently? (Answer to derive: at least 2 + any nested-relation queries; the event select failing after a successful insert = record exists but no event — another lost-event window.)

## Trace 3 (auth): API-key request from Zapier hits `GET /rest/companies`

Rows to fill: REST controller → [rest-api-find-many.handler](../../../packages/twenty-server/src/engine/api/rest/core/handlers/rest-api-find-many.handler.ts) → token validated at [jwt-auth.guard.ts#L27-L33](../../../packages/twenty-server/src/engine/guards/jwt-auth.guard.ts#L27-L33) (value shape: JWT → `{ apiKey, workspace, ... }`) → throttle *applies* (API key! [common-base-query-runner.service.ts#L352](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-base-query-runner.service.ts#L352)) → role for API key resolved via `apiKeyRoleMap` (L318-L322) → replica datasource (read-only) → RLS predicates. Risk rows: throttle rejection shape (does Zapier see 429? find the exception mapping), replica staleness.

## Trace 4 (error path): webhook receiver returns 500

| Step | File/line | What happens | What does NOT happen |
| --- | --- | --- | --- |
| 1 | [call-webhook.job.ts#L81-L88](../../../packages/twenty-server/src/engine/metadata-modules/webhook/jobs/call-webhook.job.ts#L81-L88) | axios throws (non-2xx) | — |
| 2 | [L102-L115](../../../packages/twenty-server/src/engine/metadata-modules/webhook/jobs/call-webhook.job.ts#L102-L115) | catch: audit event `success: false`, status attached | rethrow; metric increment (only success path increments L98-L101 — verify!) |
| 3 | queue | job completes "successfully" | retry, dead-letter, alert |
| 4 | tenant admin | sees failure only if they check webhook logs/audit | notification |

Fill in: what *would* each row look like if the catch rethrew? Which BullMQ options at [bullmq.driver.ts#L248](../../../packages/twenty-server/src/engine/core-modules/message-queue/drivers/bullmq.driver.ts#L248) would then matter?

---

Self-grade (per trace) — Basic: all rows filled, shapes vague. Solid: value shapes concrete at every hop (`{ name: string }`, not "the data"). Strong: at least one risk row you verified by opening a file not linked here.
