# File Reading Order

28 files. For each: why it matters, what to look for, what to ignore. Read in order — later files assume earlier ones.

## Orientation (everyone)

| # | File | Why / look for | Ignore |
| --- | --- | --- | --- |
| 1 | [CLAUDE.md](../../../CLAUDE.md) | Commands, conventions, stack | — |
| 2 | [package.json](../../../package.json) | Workspaces, `start` script, engine pins | resolutions block |
| 3 | [nx.json](../../../nx.json) | Task pipeline: what `nx build/test/lint` actually do | cache internals |
| 4 | [packages/twenty-server/src/main.ts](../../../packages/twenty-server/src/main.ts#L27-L96) | Bootstrap order: session → filters → body limits → upload middleware | SSL branch |
| 5 | [packages/twenty-server/src/app.module.ts](../../../packages/twenty-server/src/app.module.ts) | Module graph root; which subsystems exist | — |

## Junior path (locate & explain)

| # | File | Why / look for |
| --- | --- | --- |
| 6 | [workspace-schema.factory.ts](../../../packages/twenty-server/src/engine/api/graphql/workspace-schema.factory.ts#L23-L68) | Schema is *computed per workspace*: SDL from cache + generated resolvers |
| 7 | [resolver-method-names.ts](../../../packages/twenty-server/src/engine/api/graphql/workspace-resolver-builder/constants/resolver-method-names.ts) | The full vocabulary of generated operations (findMany, createOne, …) |
| 8 | [create-one-resolver.factory.ts](../../../packages/twenty-server/src/engine/api/graphql/workspace-resolver-builder/factories/create-one-resolver.factory.ts#L28-L64) | Factory returns a closure; resolver = thin adapter over a service |
| 9 | [common-create-one-query-runner.service.ts](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-create-one-query-runner.service.ts#L36-L103) | createOne = createMany of 1; `validate()` hooks |
| 10 | [common-base-query-runner.service.ts](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-base-query-runner.service.ts#L101-L159) | The pipeline. Memorize its order of operations |
| 11 | [useCreateOneRecord.ts](../../../packages/twenty-front/src/modules/object-record/hooks/useCreateOneRecord.ts#L84-L220) | Optimistic create: UUID client-side, cache write, rollback |
| 12 | [usePersistField.ts](../../../packages/twenty-front/src/modules/object-record/record-field/ui/hooks/usePersistField.ts#L61-L302) | Cell edit → typed guard chain → updateOneRecord |
| 13 | [currentWorkspaceMemberState.ts](../../../packages/twenty-front/src/modules/auth/states/currentWorkspaceMemberState.ts) | Jotai atom convention (`createAtomState`) |
| 14 | [webhook.entity.ts](../../../packages/twenty-server/src/engine/metadata-modules/webhook/entities/webhook.entity.ts) | A plain TypeORM entity in the *metadata* schema, for contrast with workspace entities |

## Mid path (modify safely)

| # | File | Why / look for |
| --- | --- | --- |
| 15 | [jwt-auth.guard.ts](../../../packages/twenty-server/src/engine/guards/jwt-auth.guard.ts#L23-L49) | Token → auth context bound to request; three principal kinds (user, apiKey, application) |
| 16 | [workspace-auth.guard.ts](../../../packages/twenty-server/src/engine/guards/workspace-auth.guard.ts#L12-L28) | Guards compose; this one only asserts a workspace exists on the request |
| 17 | [permissions.service.ts](../../../packages/twenty-server/src/engine/metadata-modules/permissions/permissions.service.ts) | Roles → object/field/settings permission resolution |
| 18 | [workspace-select-query-builder.ts](../../../packages/twenty-server/src/engine/twenty-orm/repository/workspace-select-query-builder.ts#L335-L390) | `validatePermissions()` + row-level predicates at ORM layer; `executeExistsQuery` deliberately blocked |
| 19 | [workspace-insert-query-builder.ts](../../../packages/twenty-server/src/engine/twenty-orm/repository/workspace-insert-query-builder.ts#L222-L275) | Post-insert re-select + event emission (CREATED, UPSERTED) |
| 20 | [workspace-event-emitter.ts](../../../packages/twenty-server/src/engine/workspace-event-emitter/workspace-event-emitter.ts#L39-L67) | Events are in-process EventEmitter2 batches |
| 21 | [call-webhook-jobs.job.ts](../../../packages/twenty-server/src/engine/metadata-modules/webhook/jobs/call-webhook-jobs.job.ts#L30-L75) | Fan-out job: wildcard matching, chunking, retryLimit |
| 22 | [call-webhook.job.ts](../../../packages/twenty-server/src/engine/metadata-modules/webhook/jobs/call-webhook.job.ts#L43-L117) | HMAC signing, SSRF-safe client, swallow-errors catch |
| 23 | [bullmq.driver.ts](../../../packages/twenty-server/src/engine/core-modules/message-queue/drivers/bullmq.driver.ts) | `attempts = 1 + retryLimit` (L248); queue abstraction over BullMQ vs sync driver |
| 24 | [rest-api-core.controller.ts](../../../packages/twenty-server/src/engine/api/rest/core/controllers/rest-api-core.controller.ts) | REST reuses the same runner pipeline — one brain, two mouths |

## Senior path (critique & design)

| # | File | Why / look for |
| --- | --- | --- |
| 25 | [run-workflow.job.ts](../../../packages/twenty-server/src/modules/workflow/workflow-runner/jobs/run-workflow.job.ts#L38-L133) | Status-guarded job start (re-entry safety), resume-from-step, fail → mark FAILED + rethrow |
| 26 | [field-metadata.service.ts](../../../packages/twenty-server/src/engine/metadata-modules/field-metadata/services/field-metadata.service.ts) | What "add a column at runtime" really costs: validation, migration, cache invalidation |
| 27 | [workspace-migration-runner (dir)](../../../packages/twenty-server/src/engine/workspace-manager/workspace-migration/workspace-migration-runner) | Metadata changes compiled to per-workspace DDL actions |
| 28 | [permissions.utils.ts](../../../packages/twenty-server/src/engine/twenty-orm/repository/permissions.utils.ts#L279-L360) | How join permissions are validated — the sharpest edge of the ORM boundary |

Drill: after finishing your path, close everything and draw the create-record pipeline from memory (file names optional, stages mandatory). Self-grade — Basic: 4 stages. Solid: 7 stages in order including permission check location. Strong: also mark where events are emitted and which process consumes them.
