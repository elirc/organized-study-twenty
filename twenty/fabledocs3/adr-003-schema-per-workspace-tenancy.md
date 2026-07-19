# ADR-003: Postgres schema-per-workspace tenancy with real dynamic DDL

## Status

Accepted — with an acknowledged scale ceiling (Postgres catalog growth) that
the datasource abstraction was built to eventually escape.

## Context

Given ADR-002, each workspace has a *different* set of tables and columns. The
tenancy model must therefore answer two questions at once: how tenants are
isolated, and where user-defined DDL lives.

## Decision

Each workspace gets its own **Postgres schema** in a shared database. Custom
objects become real tables; custom fields become real columns; DDL is executed
at runtime by the migration engine (ADR-004) through
`src/engine/twenty-orm/workspace-schema-manager/`. Fixed platform data (users,
workspaces, billing) lives in conventional shared-schema TypeORM entities under
`src/engine/core-modules/`. Connection management is centralized in
`src/engine/twenty-orm/global-workspace-datasource/` and
`src/engine/workspace-datasource/` so "which physical database holds this
workspace" is a resolvable question rather than a constant — a deliberate
escape hatch for future sharding.

## Alternatives considered

- **Shared tables + `workspaceId` discriminator column** — rejected as
  *incompatible with ADR-002*: tenants have different columns, so a shared
  `company` table cannot exist. Only viable by regressing to JSONB/EAV.
- **Database-per-tenant** — rejected: strongest isolation, but connection
  pools, migrations, backups, and monitoring all multiply by tenant count;
  untenable for a product with self-served workspace creation.
- **Row-level security on shared tables** — same fatal objection as the
  discriminator column, plus RLS policy management at metadata-driven scale.

## Consequences

**Positive:** real indexes, real foreign keys, real query plans on custom
data; tenant isolation is structural (a query physically targets one schema,
and cross-tenant leakage requires constructing a wrong schema name, not just
forgetting a WHERE clause); per-workspace export/delete is tractable.

**Negative:** table count grows as workspaces × objects — Postgres catalogs,
autovacuum, and `pg_dump` all degrade with very large table counts;
platform-wide upgrades need the `@RegisteredWorkspaceCommand` machinery to
iterate every workspace (O(tenants) DDL); a single shared database means noisy
neighbors share buffer cache and I/O until sharding is actually implemented.

## Evidence

`src/engine/twenty-orm/workspace-schema-manager/`;
`src/engine/twenty-orm/global-workspace-datasource/`;
`src/engine/workspace-datasource/`; the workspace-command upgrade pattern
documented in `packages/twenty-server/docs/UPGRADE_COMMANDS.md` and CLAUDE.md;
core platform entities in `src/engine/core-modules/*/` using plain TypeORM.
