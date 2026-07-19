# ADR-006: Runtime-generated GraphQL schema per workspace; REST, MCP, and OpenAPI as facades over one query pipeline

## Status

Accepted — the facade discipline has held; every new API surface (MCP is the
recent test case) was built as a translation layer, not a second data path.

## Context

ADR-002 means there is no compile-time `Company` type to write resolvers for.
Meanwhile clients (a strictly-typed React app, SDKs, Zapier, customer
integrations) want typed, introspectable, validated APIs — and the product
needs REST for integration ergonomics and OpenAPI docs.

## Decision

Two distinct GraphQL APIs with different construction models, plus facades:

1. **Metadata API** — conventional code-first NestJS GraphQL
   (`src/engine/api/graphql/metadata-graphql-api.module.ts`): metadata
   entities are compile-time-known, so ordinary resolvers work.
2. **Record ("core") API** — **generated at runtime per workspace**:
   `workspace-schema.factory.ts` builds a workspace-specific schema from
   cached metadata via `workspace-schema-builder/`, wires generic resolvers
   via `workspace-resolver-builder/`, and executes everything through one
   engine, `graphql-query-runner/` (filter parsing in
   `graphql-query-parsers/`, aggregations in `group-by/`), with cross-cutting
   extension points in `workspace-query-runner/workspace-query-hook/`.
3. **REST** (`src/engine/api/rest/` — `input-request-parsers/` +
   `rest-api.service.ts`), **MCP** (`src/engine/api/mcp/`), and generated
   **OpenAPI** docs (`src/engine/core-modules/open-api/`) all translate onto
   that same runner. None has its own data path.

## Alternatives considered

- **Generic records API** (`records(objectName, filter: JSON)`) — one static
  schema, no generation machinery; rejected because it pushes the entire type
  system onto clients: no introspection of custom objects, no codegen'd
  frontend types, no server-side filter validation.
- **Persisted/deploy-time schema generation per tenant** — rejected: metadata
  edits must be live in seconds (ADR-002's product promise).
- **Independent REST implementation** — rejected: two data paths means
  permissions and soft-delete implemented twice, guaranteed drift; the cost is
  REST semantics being "translated GraphQL" (see the 31-line
  `rest-api-exception.filter.ts` — REST error handling is deliberately thin
  because errors are already shaped upstream).

## Consequences

**Positive:** custom objects are first-class API citizens (introspection,
Apollo caching, validation); permissions/filter/soft-delete fixes land once
and cover GraphQL+REST+MCP simultaneously; invalid requests fail schema
validation instead of reaching Postgres and leaking internals.

**Negative:** the API's shape is effectively a cache entry — schema build cost
and memory per active workspace, and metadata-cache staleness becomes an API
correctness bug; REST status-code semantics are occasionally unidiomatic;
webhook payload shape is a de-facto public contract with **no versioning** —
the in-code comment in `call-webhook-jobs.job.ts` telling editors to manually
check `twenty-zapier/src/triggers/trigger_record.ts` and
`open-api/utils/computeWebhooks.utils.ts` is the entire compatibility
mechanism, which is a documented weakness of this ADR.

## Evidence

`src/engine/api/graphql/workspace-schema.factory.ts`;
`workspace-schema-builder/` and `workspace-resolver-builder/`;
`graphql-query-runner/graphql-query-parsers/`;
`src/engine/api/rest/input-request-parsers/` (note how little logic lives
there — that thinness *is* the decision); `src/engine/api/mcp/`; the Zapier
comment at the top of
`src/engine/metadata-modules/webhook/jobs/call-webhook-jobs.job.ts`.
