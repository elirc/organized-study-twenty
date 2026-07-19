# ADR-002: The data model is data — user-defined schema at runtime

## Status

Accepted — this is the foundational decision; everything in ADRs 003–006 is
downstream of it.

## Context

A CRM's competitive core is letting each customer model *their* business:
custom objects, custom fields, custom relations, per-field permissions. If the
data model is fixed at compile time, "custom field" degenerates into a bag of
generic `customField1..20` columns or a JSON blob, and every downstream feature
(filtering, sorting, aggregation, permissions, views, webhooks) treats custom
data as a second-class citizen.

## Decision

Object and field definitions are **rows, not code**. The server maintains a
metadata model (`src/engine/metadata-modules/` — `field-metadata/`,
`index-metadata/`, `data-source/`, plus ~30 `flat-*` projection modules) that
describes each workspace's objects, and all machinery that serves records —
ORM (`src/engine/twenty-orm/`), GraphQL schema
(`src/engine/api/graphql/workspace-schema.factory.ts`), REST/OpenAPI
(`src/engine/core-modules/open-api/`), permissions, views, webhooks — is
*generated or resolved from that metadata at request time*. Even the standard
objects (company, person, opportunity in `src/modules/`) are defined as
workspace entities extending
`src/engine/twenty-orm/base.workspace-entity.ts` and seeded into metadata like
any custom object: standard and custom ride identical rails.

The metadata projection each request needs is cached per workspace
(`src/engine/workspace-cache/`), versioned via
`src/engine/workspace-manager/workspace-version/`.

## Alternatives considered

- **Fixed schema + JSONB "custom fields" column** — rejected: no typed columns,
  no honest FKs, no cheap per-field indexes; custom fields would be filtered
  and aggregated worse than built-in ones, violating the "same rails" goal.
- **EAV tables** — rejected: query complexity explodes (multi-way self-joins),
  indexes are useless, and you end up reimplementing a planner in app code.
- **Code generation + redeploy per tenant** (compile the customer's model into
  artifacts) — rejected: minutes-long feedback for a spreadsheet-like editing
  experience, and a build farm as a hard runtime dependency.

## Consequences

**Positive:** customers get first-class custom objects — indexed, typed,
relational, permissioned, introspectable via GraphQL. One implementation
serves standard and custom objects, so features can't fork.

**Negative:** nothing about the data layer is knowable at compile time, which
forces a custom ORM (ADR-005), a schema generator (ADR-006), and a migration
engine (ADR-004) — a triple mortgage of bespoke infrastructure. The serving
path depends on a cached projection of the metadata, so cache staleness
becomes an API-correctness bug, not a performance bug. Debugging requires
joining "what does the metadata say" with "what does Postgres actually have."

## Evidence

`src/engine/metadata-modules/` (breadth of the module list is itself the
evidence); `src/engine/twenty-orm/base.workspace-entity.ts` and
`custom.workspace-entity.ts`; `src/modules/company/` as a standard object
defined in metadata terms; `src/engine/workspace-cache/services/workspace-cache.service.ts`
(`getOrRecompute` called from hot paths like webhook fan-out).
