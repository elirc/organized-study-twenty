# Data Model & Persistence

## The tenancy model: schema-per-workspace

Three kinds of Postgres real estate:

1. **Core/metadata schemas (shared):** users, workspaces, billing, and the metadata tables (object metadata, field metadata, roles, webhooks — e.g. [webhook.entity.ts](../../../packages/twenty-server/src/engine/metadata-modules/webhook/entities/webhook.entity.ts) is a classic TypeORM entity here).
2. **Per-workspace schemas (tenant data):** each workspace gets its own schema containing *real columns* for its objects — including custom ones. DDL is managed by [workspace-schema-manager](../../../packages/twenty-server/src/engine/twenty-orm/workspace-schema-manager) and the workspace-migration machinery.
3. **Redis:** metadata caches (flat maps), sessions, queues. **ClickHouse (optional):** analytics.

Interview framing — the three ways to store user-defined fields:

| Approach | Pros | Cons |
| --- | --- | --- |
| EAV table (rows of key/value) | trivial to add fields | queries become joins-on-joins; no types, no indexes |
| JSONB column | flexible, indexable-ish | weak typing, awkward relations, migration-by-convention |
| **Dynamic DDL (Twenty's choice)** | real columns: types, indexes, FKs, fast queries | schema changes at runtime = the hardest subsystem in the repo; per-tenant schema drift risk |

Twenty accepted operational complexity to keep relational integrity and query performance. When asked "how would you support custom fields?", presenting this table then defending one row *is* the mid-to-senior answer.

## Entities twice over

Standard objects are declared as `*.workspace-entity.ts` classes (e.g. [workflow-run.workspace-entity.ts](../../../packages/twenty-server/src/modules/workflow/common/standard-objects/workflow-run.workspace-entity.ts)) extending the base at [base.workspace-entity.ts](../../../packages/twenty-server/src/engine/twenty-orm/base.workspace-entity.ts) — but these are *metadata source code*, synced into each workspace's metadata at upgrade time, not conventional TypeORM entities. Custom objects use [custom.workspace-entity.ts](../../../packages/twenty-server/src/engine/twenty-orm/custom.workspace-entity.ts) as their runtime stand-in. The **flat maps** (denormalized metadata snapshots in Redis, threaded through every query — [common-base-query-runner.service.ts#L105-L110](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-base-query-runner.service.ts#L105-L110)) are the working representation.

## Migrations: three distinct mechanisms (do not confuse)

| Mechanism | Scope | Written by | Example trigger |
| --- | --- | --- | --- |
| Core TypeORM migrations | shared schemas | hand | new billing table |
| **Workspace migrations** | each tenant schema | generated from metadata diffs ([workspace-migration-builder](../../../packages/twenty-server/src/engine/workspace-manager/workspace-migration/workspace-migration-builder), executed by [workspace-migration-runner action handlers](../../../packages/twenty-server/src/engine/workspace-manager/workspace-migration/workspace-migration-runner/action-handlers)) | user adds a field |
| **Instance commands** | whole installation, iterating workspaces | hand, with `up`/`down`, fast (schema) or slow (data backfill) — see [CLAUDE.md](../../../CLAUDE.md) and `packages/twenty-server/docs/UPGRADE_COMMANDS.md` | version upgrade changes a standard object |

"How to safely change the schema" here = change the workspace-entity class → generate an instance command (`npx nx run twenty-server:database:migrate:generate --name <name> --type <fast|slow>`, __inferred__) → include `up` *and* `down` → never rewrite committed command logic. Rollback is a first-class deliverable, not an afterthought — say that sentence in interviews.

## Transactions and consistency expectations

- Single-record writes ride single statements plus a post-write re-select ([workspace-insert-query-builder.ts#L224-L246](../../../packages/twenty-server/src/engine/twenty-orm/repository/workspace-insert-query-builder.ts#L224-L246)).
- The entity manager and query builders support TypeORM transactions; **whether a given flow is transactional is a per-call-site fact** — verify before claiming. The event-emission-vs-commit question is analyzed in [04-side-effects-async-and-reliability.md](04-side-effects-async-and-reliability.md).
- Reads may hit a **replica** for read-only runners ([common-base-query-runner.service.ts#L332-L334](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-base-query-runner.service.ts#L332-L334)) — read-your-writes is not guaranteed across requests; the front papers over gaps with optimistic cache writes.
- Soft delete is a first-class state (`deleteOne` vs `destroyOne` resolver factories; a dedicated [workspace-soft-delete-query-builder.ts](../../../packages/twenty-server/src/engine/twenty-orm/repository/workspace-soft-delete-query-builder.ts)); queries must consistently respect `deletedAt`, and there's an integration test pinning the tricky case ([soft-deleted-relation.integration-spec.ts](../../../packages/twenty-server/test/integration/graphql/suites/soft-deleted-relation.integration-spec.ts)).

## Indexes

Index metadata is itself metadata ([engine/metadata-modules/index-metadata](../../../packages/twenty-server/src/engine/metadata-modules/index-metadata)) and materialized per workspace schema — meaning index strategy is programmable per tenant object. When you investigate slow queries, check what indexes the metadata *says* exist before reading Postgres.

## Drills

1. Write (don't run) the plan for adding a required `industry` field to Company across 10k workspaces: which of the three migration mechanisms, what's the backfill story, what's `down`? Self-grade — Strong: you chose a *slow* instance command, defaulted the column before enforcing anything, and your `down` is data-safe.
2. Find how `deletedAt` filtering enters an ordinary findMany. Start at [workspace-soft-delete-query-builder.ts](../../../packages/twenty-server/src/engine/twenty-orm/repository/workspace-soft-delete-query-builder.ts). Strong: you can say what breaks if one code path forgets it (deleted rows leak into lists — a *consistency invariant* violation).

## Interview angle

- "Design a multi-tenant SaaS database" → schema-per-tenant vs row-scoping vs db-per-tenant; Twenty is your worked example of the middle path. ([08-interview-prep/03](../08-interview-prep/03-api-and-data-modeling-questions.md) Q1–Q3)
- "How do you run a migration with zero downtime?" → expand/contract + the fast/slow command distinction. (Q6)
