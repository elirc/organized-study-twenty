# Mid-Level Feature Tickets (10)

Cross-layer work. **Rule: write a half-page design note (approach, alternatives, risk, rollback) before any code.** Each ticket lists: scope, anchors, design questions your note must answer, risk & rollback, tests, and interview story potential.

---

## Ticket M1: Webhook retry with backoff (behind a flag)
Medium-Hard, 2–4 days. The flagship. Rethrow on retryable failures (5xx/timeout, not 4xx) in [call-webhook.job.ts#L102-L116](../../../packages/twenty-server/src/engine/metadata-modules/webhook/jobs/call-webhook.job.ts#L102-L116) so BullMQ `attempts` ([bullmq.driver.ts#L248](../../../packages/twenty-server/src/engine/core-modules/message-queue/drivers/bullmq.driver.ts#L248)) engages, gated by a feature flag ([engine/core-modules/feature-flag](../../../packages/twenty-server/src/engine/core-modules/feature-flag)).
Design questions: chunk problem — the job carries up to 20 deliveries ([call-webhook-jobs.job.ts#L66-L74](../../../packages/twenty-server/src/engine/metadata-modules/webhook/jobs/call-webhook-jobs.job.ts#L66-L74)); a rethrow retries *all 20*, re-delivering to receivers that succeeded. Your note must solve this (per-delivery jobs? partial-failure tracking in job state? idempotent receivers via nonce?). Audit semantics per attempt? Backoff curve?
Risk & rollback: duplicate deliveries to consumers (Zapier!); flag-off restores current behavior instantly.
Tests: unit (Recipe 4 extended), integration with a flaky fake receiver.
Interview story potential: a complete at-least-once vs at-most-once design decision with a migration path — the single best story in this curriculum.

## Ticket M2: Per-workspace webhook delivery dashboard data
Medium, 2–3 days. Expose webhook delivery stats (success/fail counts from audit events) via a resolver for the settings UI. Layers: audit query service → GraphQL (metadata API, guarded by settings permission — follow [settings-permission.guard.ts](../../../packages/twenty-server/src/engine/guards/settings-permission.guard.ts) precedent) → settings page section.
Design questions: query cost over audit volume; pagination; cache TTL.
Rollback: additive API — remove resolver.
Interview story: "read-path feature over an event-sourced audit trail."

## Ticket M3: "Duplicate record" action
Medium, 3–5 days. Table row action: duplicate a record (minus unique/composite conflicts). Layers: front action menu → `useCreateOneRecord` with cloned sanitized input ([sanitizeRecordInput](../../../packages/twenty-front/src/modules/object-record/utils/sanitizeRecordInput.ts)) → nothing server-side (reuse createOne).
Design questions: which fields *shouldn't* copy (createdBy, position, attachments?); relation handling depth; the find-duplicates feature's interaction ([find-duplicates runner](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-find-duplicates-query-runner.service.ts) will flag your copy — feature, not bug?).
Interview story: "shipped a feature with zero new server surface by understanding the existing API."

## Ticket M4: Rate-limit visibility header
Medium, 2–3 days. Return remaining-tokens headers on REST responses for API-key traffic (read from the throttler at [common-base-query-runner.service.ts#L350-L395](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-base-query-runner.service.ts#L350-L395)).
Design questions: does the token bucket expose remaining counts cheaply? Header naming (industry: `X-RateLimit-*`); GraphQL story (extensions field?) — REST-only first slice.
Rollback: headers are additive.
Interview story: "API ergonomics — turning a silent 429 into a self-describing contract."

## Ticket M5: Bulk-create front path for spreadsheet import (verify first)
Medium-Hard, 3–5 days. If import uses per-record creates (verify what [spreadsheet-import](../../../packages/twenty-front/src/modules/spreadsheet-import) calls — [useBatchCreateManyRecords](../../../packages/twenty-front/src/modules/object-record/hooks/useBatchCreateManyRecords.ts) exists), route it through batched createMany with progress UI.
Design questions: batch size vs mutation payload limits ([main.ts#L66-L70](../../../packages/twenty-server/src/main.ts#L66-L70) body limits); partial-failure UX (which rows failed); aggregate refetch once, not per record.
Interview story: "10x'd an import by moving from N mutations to ceil(N/k) — with measurements."

## Ticket M6: Workflow run start compare-and-set
Medium-Hard, 2–4 days. Replace read-then-act status guard ([run-workflow.job.ts#L90-L95](../../../packages/twenty-server/src/modules/workflow/workflow-runner/jobs/run-workflow.job.ts#L90-L95)) with an atomic conditional UPDATE via the workspace service.
Design questions: exact SQL through twenty-orm's update builder; behavior when CAS loses (log + return, not throw?); does resume need the same treatment (L149-L151)?
Tests: two concurrent starters against a real DB — integration.
Interview story: "closed a race with compare-and-set and proved it with a concurrency test."

## Ticket M7: Field-level "last edited by" tooltip
Medium, 3–5 days. Show who last edited a field value, from the timeline/audit data ([modules/timeline](../../../packages/twenty-server/src/modules/timeline)). Layers: existing event data → resolver → cell hover UI.
Design questions: is per-field granularity actually in the event diffs (read [transform-event-to-webhook-event.ts](../../../packages/twenty-server/src/engine/metadata-modules/webhook/utils/transform-event-to-webhook-event.ts) sibling utils for diff shape)? Cost of per-cell lookups (batch by record).
Interview story: "built UX on top of an existing event stream instead of new writes."

## Ticket M8: Standard object for "Deal loss reason" (metadata exercise)
Medium, 2–4 days *in a dev workspace*. Add a select field to Opportunity as a *standard* field: workspace-entity change + generated instance command (fast) + front default view update.
Design questions: standard-object sync mechanics; up/down; why NOT just a custom field (answer: product default for all workspaces).
Rollback: `down` logic mandatory.
Interview story: "runtime-schema systems still need code-defined defaults — I shipped one end to end."

## Ticket M9: REST parity test matrix
Medium, 3–4 days. Table-driven integration suite asserting GraphQL and REST return equivalent data/errors for find/create/update/delete on one object (handlers in [rest/core/handlers](../../../packages/twenty-server/src/engine/api/rest/core/handlers) vs resolver factories).
Design questions: canonicalizing response shapes for comparison; where divergence is *intended* (pagination style?).
Interview story: "encoded 'one pipeline, two transports' as an executable contract."

## Ticket M10: Permission-denied UX for restricted fields
Medium, 3–5 days. When field-level permissions restrict a column, the table should show a lock state rather than empty values. Layers: object permissions on front ([useObjectPermissions](../../../packages/twenty-front/src/modules/object-record/hooks/useObjectPermissions.ts)) → cell display fallback.
Design questions: can the client distinguish "null value" from "restricted" in current payloads? (If not, the honest ticket is bigger — your design note should say so and slice it.)
Interview story: "security features need UX or users file data-loss bugs."

---

Common review bar: design note first; migration/rollback stated; the denied/failure case tested with the same care as success; blast radius one module unless justified.
