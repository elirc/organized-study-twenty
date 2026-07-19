# ADR-005: A custom ORM layer with permissions and soft-delete enforced in the query builders

## Status

Accepted — enforced by convention and review; the escape hatch (raw datasource
access) still exists for system paths, so the guarantee is strong but not
mechanical.

## Context

TypeORM requires compile-time entities; ADR-002 makes entities runtime data.
Additionally, records are served through *four* API surfaces (GraphQL, REST,
MCP, workflows/jobs), and every one must apply identical row/field permissions
and soft-delete filtering. Enforcing those rules in each resolver/service
means four chances per feature to forget one.

## Decision

Build **twenty-orm** (`src/engine/twenty-orm/`): a data layer that speaks
TypeORM's dialect but resolves table/column shape from cached workspace
metadata at request time, and — the philosophically important part — pushes
cross-cutting invariants *into the query construction itself*:

- `repository/workspace.repository.ts` (~980 lines) is the single sanctioned
  entry point for record reads/writes.
- Per-operation query-builder subclasses
  (`workspace-select-query-builder.ts`, `-insert-`, `-update-`, `-delete-`,
  `-soft-delete-query-builder.ts`) inject schema resolution, permission
  predicates (`repository/permissions.utils.ts`), and default
  `deletedAt IS NULL` filtering — including on joined relations — so a caller
  cannot forget them.
- Postgres errors are translated to typed exceptions before escaping
  (`twenty-orm/error-handling/`, `exceptions/twenty-orm.exception.ts`).

## Alternatives considered

- **Raw TypeORM + permission checks in services** — the conventional layering;
  rejected because "every caller remembers" across four API surfaces and a
  runtime-defined schema is a losing bet; a single forgotten check is a
  cross-tenant or deleted-data leak.
- **Postgres RLS for permissions** — rejected: policies would have to be
  generated and migrated per metadata change on top of ADR-004's already
  nontrivial DDL engine; debugging authz inside the database at this dynamism
  level is worse than in application code.
- **Query-by-string internal API** (skip ORM semantics entirely) — rejected:
  loses typed query construction and makes the permission-injection point
  diffuse.

## Consequences

**Positive:** authorization and soft-delete are properties of the *data layer*,
uniform across GraphQL/REST/MCP/jobs by construction; new API surfaces (MCP
arrived late) inherited correct authz for free; Postgres internals never leak
to clients raw.

**Negative:** a large bespoke ORM is a maintenance liability owned by the team,
not the ecosystem; TypeORM upgrades are risky because the builders subclass
its internals; the raw-datasource escape hatch means the invariant is
ultimately social ("go through WorkspaceRepository") — the pattern of adding
missing batch capabilities *to* the repository rather than around it is
visible in its size; soft-delete-on-joins produces subtle aggregation
semantics that surface as "dashboard number is wrong" bugs.

## Evidence

`src/engine/twenty-orm/repository/` (all seven files);
`repository/permissions.utils.ts`; `twenty-orm/entity-manager/` and
`factories/` for metadata-to-repository construction;
`src/engine/trash-cleanup/` for the soft-delete lifecycle's other end.
