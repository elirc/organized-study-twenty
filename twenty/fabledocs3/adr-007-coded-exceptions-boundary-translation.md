# ADR-007: Coded domain exceptions, translated to protocol semantics only at API boundaries

## Status

Accepted — consistently applied in newer modules; older corners still throw
bare `Error`, so enforcement is directional rather than complete.

## Context

The same domain failure ("field does not exist," "permission denied,"
"migration step failed") must surface through GraphQL (error extensions, no
HTTP status in-band), REST (status codes), MCP, and background-job logs. If
domain code throws protocol-shaped exceptions (NestJS's
`BadRequestException` et al.), every non-HTTP surface must *un*-translate and
re-translate, and Postgres errors risk reaching clients verbatim.

## Decision

Three tiers:

1. **Domain code throws typed, coded exceptions owned by each module** —
   `twenty-orm/exceptions/twenty-orm.exception.ts`,
   `workspace-manager/workspace-migration.exception.ts`,
   `api/graphql/workspace-query-runner/workspace-query-runner.exception.ts`,
   and siblings. Raw Postgres errors are converted inside the data layer
   (`twenty-orm/error-handling/`) before they can escape.
2. **Boundaries translate codes to protocol semantics** — REST via the
   deliberately thin `src/engine/api/rest/rest-api-exception.filter.ts` and
   `api/rest/errors/`; GraphQL via hooks in the exception-handler module.
3. **Reporting is centralized behind a driver interface** —
   `src/engine/core-modules/exception-handler/exception-handler.service.ts`
   with `drivers/sentry.driver.ts` and `drivers/console.driver.ts` (see
   ADR-009), so "where do exceptions go" is deployment configuration.
   A last-resort `src/filters/unhandled-exception.filter.ts` exists mostly to
   re-attach CORS headers — its header comment records the real-world failure
   (pre-CORS-middleware exceptions produce responses browsers misreport as
   CORS errors) that motivated it.

## Alternatives considered

- **Idiomatic NestJS HTTP exceptions from services** — rejected: couples
  domain logic to one of four protocols; convenient and wrong.
- **Result/Either types instead of exceptions** — no evidence it was tried;
  would fight NestJS's filter/interceptor model that the boundary translation
  relies on.
- **Global mapping table from Postgres error codes to client errors** —
  rejected in favor of translating *inside* the data layer, where metadata
  context exists to say *which field's* constraint failed.

## Consequences

**Positive:** one domain error definition serves all four surfaces; clients
get stable machine-readable codes rather than paraphrased Postgres text;
internals (schema names, constraint names) don't leak; swapping/adding error
sinks is config.

**Negative:** per-module exception classes and code enums are ceremony, and
older code paths that predate the discipline still throw bare `Error` —
meaning the catch-all filter must handle genuinely unknown shapes; boundary
translation means REST statuses are sometimes lawyerly (400 where a purist
wants 422); in background jobs there is no protocol boundary at all, so the
throw/catch decision silently becomes a *delivery-semantics* decision — the
sharpest consequence of which is ADR-010.

## Evidence

The exception files named above; `src/engine/api/rest/errors/`;
`src/engine/core-modules/exception-handler/` (module, service, `drivers/`,
`hooks/`, `interfaces/`); `src/filters/unhandled-exception.filter.ts` and its
two-sentence war-story comment.
