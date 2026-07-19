# 04 — SQL & tenancy lab: reading the multi-tenant database with your own eyes

Every serious debugging session here ends in Postgres. This lab makes the
ADR-003 tenancy model concrete by having you *look at it*. Do it once slowly;
you'll reuse the queries forever.

**Setup:** `bash packages/twenty-utils/setup-dev-env.sh`, then
`npx nx database:reset twenty-server` for a seeded instance
(`src/engine/workspace-manager/dev-seeder/` controls what you get). Connect
with `psql` — or use the read-only Postgres MCP server configured in
`.mcp.json` (per CLAUDE.md, it exists exactly for this kind of inspection).

Answers aren't provided — the database is the answer key. Verify everything.

## Part 1 — Map the schemas (20 min)

```sql
SELECT schema_name FROM information_schema.schemata ORDER BY 1;
```

1. Identify the three kinds of schemas you see: platform/core tables,
   metadata tables, and per-workspace schemas. What naming convention marks a
   workspace schema, and what does the suffix encode?
2. Pick the seeded workspace's schema. List its tables. Match five of them to
   entity definitions under `src/modules/` (e.g., company, person,
   opportunity — the `base.workspace-entity.ts` descendants).
3. In the *metadata* schema, find the tables describing objects and fields.
   Write the join that answers: "for workspace W, list every object with its
   field count." Keep this query; it's your Rosetta Stone.

## Part 2 — Watch metadata become DDL (30 min)

With the app running (`yarn start`), in the Settings UI create a custom
object `Warehouse` with a custom SELECT field `region`.

4. Re-run your Part 1 queries. Find: the new metadata rows, the new table in
   the workspace schema, and the Postgres **enum type** (or check constraint —
   verify which!) backing the SELECT field.
5. Diff reality against expectation: does the table have columns you didn't
   create? List the bookkeeping columns every workspace table gets (id,
   timestamps, `deletedAt`, position…) and match them to
   `base.workspace-entity.ts`.
6. Find the index metadata (`index-metadata` module server-side) and confirm
   the actual Postgres indexes on your new table
   (`\di` or `pg_indexes`). Metadata and catalog should agree — this
   agreement is exactly what ADR-004's diff engine maintains, and you're
   looking at its output.

## Part 3 — Soft delete with your own eyes (20 min)

In the UI, delete a seeded company (to trash).

7. Find the row. What changed? (`deletedAt` — confirm, don't assume.)
8. Query the company's opportunities through their foreign key. The rows
   still reference the trashed company. Now explain, in SQL terms, what the
   `workspace-select-query-builder` must add to *every join* to keep trashed
   partners invisible — and write the naive join that gets it wrong (this is
   Drill 3's bug class, from the database's point of view).
9. Restore from trash in the UI; confirm the column reverts.

## Part 4 — Tenancy isolation, adversarially (20 min)

10. The seeder may create multiple workspaces (check). Write the query a
    *bug* would need to leak data across tenants. Observe that it requires
    naming another workspace's schema explicitly — there is no shared table
    where a missing `WHERE workspaceId = ?` leaks rows. That structural
    property is ADR-003's core claim; you've now verified it empirically.
11. Counterpoint: name two things this isolation does *not* protect
    (shared buffer cache/IO — noisy neighbors; and any code path that
    constructs schema names from unvalidated input). Where in the server
    would you look for schema-name construction? (Start:
    `src/engine/twenty-orm/global-workspace-datasource/` and
    `workspace-schema-manager/`; note the existence of
    `core-modules/sql-sanitization/`.)

## Part 5 — The catalog-bloat ceiling (15 min)

12. Count tables per workspace schema. Multiply by a hypothetical 10,000
    workspaces. Query `pg_class` for total relation count and reflect on
    ADR-003's "scale ceiling" consequence — you now have the number that
    makes it concrete.
13. Closing question, in writing: if sharding workspaces across physical
    databases becomes necessary, why does the existence of
    `global-workspace-datasource` mean query-layer code shouldn't need to
    change? What *would* need to change?

## Exit criteria

You can, without notes: name the three schema kinds; write the
object-with-field-count metadata join; explain soft-delete join semantics in
SQL; and state precisely why cross-tenant leakage requires a constructed
schema name rather than a forgotten predicate.
