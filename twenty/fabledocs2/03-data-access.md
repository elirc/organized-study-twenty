# 03 — Data access: twenty-orm, or why we forked TypeORM's brain

## What it is and why it exists

TypeORM wants entities known at compile time. Our entities are rows in a metadata
table. So we built **twenty-orm** (`src/engine/twenty-orm/`) — a layer that speaks
TypeORM's dialect but resolves entity shape, table names, column names, relations,
permissions, and soft-deletion **at request time from workspace metadata**.

The pieces:

- `repository/workspace.repository.ts` — the ~980-line heart. Every record read
  and write in the product goes through this class or the query builders next to it.
- `repository/workspace-select-query-builder.ts`, `workspace-insert-query-builder.ts`,
  `workspace-update-query-builder.ts`, `workspace-delete-query-builder.ts`,
  `workspace-soft-delete-query-builder.ts` — TypeORM query-builder subclasses that
  inject workspace-schema resolution, permission predicates, and soft-delete
  filtering into every query, so callers can't forget them.
- `repository/permissions.utils.ts` — permission checks living *inside* the data
  layer, not sprinkled through resolvers.
- `global-workspace-datasource/` + `src/engine/workspace-datasource/` — connection
  and datasource management across many workspace schemas without a pool per tenant.
- `workspace-schema-manager/` — the DDL executor twenty-orm exposes to the
  migration runner (doc 01).
- Standard objects (person, company, opportunity, notes, tasks…) are defined in
  code as **workspace entities** — see `src/modules/` (e.g. `src/modules/company/`,
  `src/modules/person/`) extending `base.workspace-entity.ts` / synced into
  metadata like any custom object. Standard and custom objects ride the same rails;
  standard ones are just pre-seeded metadata.

## The hard decisions

### 1. Schema-per-workspace vs. `workspaceId` column vs. database-per-tenant

- **Database-per-tenant**: strongest isolation, operational nightmare at our
  workspace count (connection pools, migrations, backups × N databases). Rejected.
- **Shared tables + `workspaceId` column**: simplest ops, but remember doc 01 —
  workspaces have *different columns*. A shared `company` table can't hold one
  tenant's 60 custom fields and another's 3. You end up back at EAV or JSONB.
  Rejected as incompatible with dynamic DDL.
- **Postgres schema per workspace**: real tables per tenant, one database, one
  pool, `search_path`-free explicit schema qualification. Chosen.

The tradeoff that still bothers me: Postgres catalogs are not designed for
hundreds of thousands of tables. Catalog bloat is real; `information_schema`
queries slow down; autovacuum scheduling gets noisy. We monitor it (doc 06) and
it's fine at current scale, but there is a scale ceiling on this design and we
will meet it before the company dies of anything else. The eventual answer is
probably sharding workspaces across database clusters — the
`global-workspace-datasource` abstraction was deliberately built so "which
physical database" can become a lookup later.

### 2. Enforce permissions and soft-delete in the query builder, not in services

The rejected alternative was the conventional one: services check permissions,
repositories are dumb. We inverted it — `workspace-select-query-builder.ts` and
friends refuse to build a query without the permission context, and
`permissions.utils.ts` computes the restricted field/row sets at build time.

Why: with a runtime-defined schema and four API surfaces (GraphQL, REST, MCP,
workflows), "every service remembers to check" is a bet against entropy with
terrible odds. Putting it in the builder means the *type system* makes the unsafe
path awkward and code review makes it visible.

What still bothers me: the escape hatch exists. Raw datasource access is
necessary for the migration runner and a few system paths, and there's no
mechanical guardrail — no lint rule strong enough — that stops a feature developer
from grabbing a raw TypeORM manager "just for this one query." The rule is social:
**if you're not going through `WorkspaceRepository`, your PR needs my sign-off.**
See the incident below for why I'm humorless about this.

### 3. Soft delete as a first-class query-builder concern

CRM users delete things constantly and regret it constantly. Soft-delete (a
`deletedAt` column, filtered by default) was table stakes. The decision was
*where* to implement it. TypeORM has native soft-delete support, but it doesn't
survive our dynamic-entity resolution, and its default-filtering is easy to
bypass accidentally via query builder joins.

So `workspace-soft-delete-query-builder.ts` exists, and the select builder
applies the `deletedAt IS NULL` predicate to the root *and to every joined
relation*. The "trash" feature (and `src/engine/trash-cleanup/` for hard-purging
old soft-deleted rows) sits on top.

Residual annoyance: aggregations. `COUNT` semantics with soft-deleted rows across
joined relations produce subtle off-by-a-few bugs that unit tests don't catch
because the interplay only shows up with real SQL. This is a standing source of
"the dashboard number is wrong" tickets — check soft-delete join predicates
*first* when you get one.

## Evidence to go read

- `src/engine/twenty-orm/repository/workspace.repository.ts` — read the whole
  thing once, slowly. It's the most load-bearing file in the repo.
- `src/engine/twenty-orm/repository/permissions.utils.ts` — how row/field
  restriction becomes SQL.
- `src/engine/twenty-orm/entity-manager/` and `factories/` — how a "repository
  for object X in workspace Y" gets constructed from cached metadata.
- `src/engine/twenty-orm/error-handling/` and `exceptions/twenty-orm.exception.ts`
  — how Postgres errors are translated before they escape (ties into doc 05).
- `src/modules/company/` — a standard object defined as a workspace entity;
  compare with what lands in metadata after sync.

## A production incident this design caused (and the rule it created)

The "ghost records" incident. A background feature (early version of a dedupe
job) used a raw datasource query to bulk-update person records — bypassing
`WorkspaceRepository` because the builder API didn't yet support the batch shape
it needed. The raw update ignored the soft-delete predicate and resurrected
records users had deleted: it cleared fields on rows where `deletedAt` was set,
and a subsequent restore-from-trash surfaced half-blanked contacts. Support tickets
described it, accurately, as "the CRM ate my notes."

Cleanup required replaying audit events (doc 06 — this is where the audit trail
earned its storage bill) to reconstruct field values. The permanent fixes: the
batch APIs the job actually needed were added to `WorkspaceRepository`, and the
social rule from decision #2 became policy. When the safe path is missing a
capability, we extend the safe path. We do not route around it.

## Before you move on, you should be able to answer:

1. Why is schema-per-workspace the only one of the three tenancy options
   compatible with the dynamic DDL decision from doc 01?
2. Where exactly does a permission restriction become part of the SQL, and why
   was that put in the query-builder layer instead of resolvers or services?
3. You need a bulk operation `WorkspaceRepository` doesn't support. What are you
   required to do, and what are you required *not* to do?
