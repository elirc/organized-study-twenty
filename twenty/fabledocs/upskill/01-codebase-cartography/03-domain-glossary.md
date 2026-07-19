# Domain Glossary

Two vocabularies coexist: the **CRM domain** (companies, people, opportunities) and the **metadata platform** (objects, fields, views). Confusing them is the #1 junior mistake in this repo.

## Metadata platform terms

| Term | Meaning | Code location |
| --- | --- | --- |
| **Workspace** | A tenant. Owns its own metadata and its own Postgres schema. | [engine/core-modules/workspace/](../../../packages/twenty-server/src/engine/core-modules/workspace) |
| **Object metadata** | Definition of a record type (standard like `company`, or custom). The GraphQL schema is generated from these. | [engine/metadata-modules/object-metadata/](../../../packages/twenty-server/src/engine/metadata-modules/object-metadata) |
| **Field metadata** | Definition of one field on an object, including composite types (FULL_NAME, CURRENCY, EMAILS…). | [engine/metadata-modules/field-metadata/](../../../packages/twenty-server/src/engine/metadata-modules/field-metadata) |
| **Flat entity / flat maps** | Denormalized, cache-friendly snapshots of metadata (`FlatObjectMetadata`, `flatFieldMetadataMaps`) passed through the query pipeline instead of live DB entities. | [engine/metadata-modules/flat-entity/](../../../packages/twenty-server/src/engine/metadata-modules/flat-entity); used throughout [common-base-query-runner.service.ts](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-base-query-runner.service.ts#L105-L110) |
| **Workspace schema** | The per-tenant Postgres schema holding actual record tables. Contrast with the shared `core`/metadata schema. | [engine/twenty-orm/workspace-schema-manager/](../../../packages/twenty-server/src/engine/twenty-orm/workspace-schema-manager) |
| **Workspace migration** | A generated set of DDL actions applied when metadata changes (create table, add column…). Not a hand-written TypeORM migration. | [engine/workspace-manager/workspace-migration/](../../../packages/twenty-server/src/engine/workspace-manager/workspace-migration) |
| **Instance command** | Hand-written upgrade command for the *whole installation* (fast = schema, slow = data backfill). See [CLAUDE.md](../../../CLAUDE.md) "Database & Upgrade Commands". | `packages/twenty-server/src/database/commands` |
| **Standard object** | Built-in object shipped as a `*.workspace-entity.ts` class, synced into every workspace's metadata. | e.g. [workflow-run.workspace-entity.ts](../../../packages/twenty-server/src/modules/workflow/common/standard-objects/workflow-run.workspace-entity.ts) |
| **Query runner** | The service that actually executes a CRUD operation after args processing. | [engine/api/common/common-query-runners/](../../../packages/twenty-server/src/engine/api/common/common-query-runners) |
| **Pre/post query hooks** | Per-object interception points around every operation. | invoked at [common-base-query-runner.service.ts](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-base-query-runner.service.ts#L205-L211) |
| **Role / permission flag / object permission / RLS predicate** | Authorization model layers: workspace-settings flags, per-object CRUD grants, row-level predicates. | [engine/metadata-modules/permissions/](../../../packages/twenty-server/src/engine/metadata-modules/permissions), [twenty-orm/repository/permissions.utils.ts](../../../packages/twenty-server/src/engine/twenty-orm/repository/permissions.utils.ts) |
| **View** | Saved table/board configuration (filters, sorts, groups) — metadata, not records. | [engine/metadata-modules/flat-view/](../../../packages/twenty-server/src/engine/metadata-modules/flat-view) |

## CRM domain terms

| Term | Meaning | Code location |
| --- | --- | --- |
| **Company / Person / Opportunity / Note / Task** | Standard CRM objects | [modules/company/](../../../packages/twenty-server/src/modules/company) etc. |
| **Workspace member** | A user *within* a workspace (profile, settings). Distinct from `User` (login identity) and `UserWorkspace` (join). | [modules/workspace-member/](../../../packages/twenty-server/src/modules/workspace-member) |
| **Connected account** | An external mailbox/calendar connection (Google/Microsoft/IMAP). | [modules/connected-account/](../../../packages/twenty-server/src/modules/connected-account) |
| **Message channel / message import** | Email sync machinery. | [modules/messaging/](../../../packages/twenty-server/src/modules/messaging) |
| **Workflow / workflow version / workflow run** | Automation: trigger + steps; versions are immutable, runs track execution state. | [modules/workflow/](../../../packages/twenty-server/src/modules/workflow) |
| **Webhook** | Tenant-configured HTTP callback on record events, with wildcard subscriptions like `*.created`. | [engine/metadata-modules/webhook/](../../../packages/twenty-server/src/engine/metadata-modules/webhook) |

## Confusables

- **`/graphql` vs `/metadata`** — two GraphQL endpoints: records vs schema-definition. Both set up in [main.ts](../../../packages/twenty-server/src/main.ts#L74-L88).
- **Workspace migration vs instance command** — per-tenant generated DDL vs installation-wide hand-written upgrade.
- **`User` vs `UserWorkspace` vs `WorkspaceMember`** — identity vs membership join vs in-tenant profile record.
- **Object permission vs settings permission** — CRUD on records vs admin capabilities; settings checks happen in the pipeline at [common-base-query-runner.service.ts](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-base-query-runner.service.ts#L274-L311).
- **Soft delete vs destroy** — `deleteOne` soft-deletes (restorable), `destroyOne` is permanent; both have separate resolver factories in [workspace-resolver-builder/factories/](../../../packages/twenty-server/src/engine/api/graphql/workspace-resolver-builder/factories).

Drill: without looking, explain to a rubber duck why "add a field to Company" does **not** involve writing a TypeORM migration in this repo, and what happens instead. Self-grade — Basic: mentions field metadata. Solid: mentions workspace migration generation + schema cache invalidation. Strong: contrasts with a classic Prisma/TypeORM app and names the tradeoff (runtime flexibility vs static type safety).
