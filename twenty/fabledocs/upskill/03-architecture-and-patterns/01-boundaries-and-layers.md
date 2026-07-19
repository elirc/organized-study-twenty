# Boundaries & Layers

## The server's layer stack

```
Transport      GraphQL resolvers / REST handlers / MCP     (engine/api/graphql, engine/api/rest)
   │  owns: protocol shapes, arg extraction. must not own: business rules, SQL
Pipeline       Common query runners                        (engine/api/common)
   │  owns: validation, throttling, hooks, permission gates, arg coercion
Domain hooks   Pre/post query hooks + feature modules      (modules/*)
   │  owns: product behavior (CRM rules, sync, workflows)
Data access    twenty-orm query builders + repositories    (engine/twenty-orm)
   │  owns: SQL generation, row/field permission enforcement, event emission
Storage        Postgres (per-workspace schemas), Redis
```

A **boundary** is where responsibility and data shape change. The strongest boundary decision in this repo: GraphQL and REST are *both* thin adapters over the same pipeline. Compare [create-one-resolver.factory.ts#L33-L63](../../../packages/twenty-server/src/engine/api/graphql/workspace-resolver-builder/factories/create-one-resolver.factory.ts#L33-L63) with the REST handlers in [rest-api-core.controller.ts](../../../packages/twenty-server/src/engine/api/rest/core/controllers/rest-api-core.controller.ts) — different mouths, one brain. Consequence: a permission fix in the pipeline fixes both APIs at once. That's the property to look for in any codebase: *is authorization on the shared path or duplicated per transport?*

## Good boundaries, with evidence

- **Enforcement at the lowest layer.** Object/row/field permissions are validated inside the query builders themselves ([workspace-select-query-builder.ts#L342-L352](../../../packages/twenty-server/src/engine/twenty-orm/repository/workspace-select-query-builder.ts#L342-L352)), so feature code physically cannot forget them. The type system even blocks dangerous methods: `executeExistsQuery` throws `METHOD_NOT_ALLOWED` because it would sidestep the permission path ([L335-L340](../../../packages/twenty-server/src/engine/twenty-orm/repository/workspace-select-query-builder.ts#L335-L340)). Making the wrong thing *impossible* beats documenting it.
- **Platform vs product.** `engine/` knows nothing about CRM; `modules/` defines companies and workflows *as data plus hooks*. You could ship a different product on the same engine. The test: [modules/company/](../../../packages/twenty-server/src/modules/company) contains almost no query code.
- **External contract quarantined.** The webhook payload shape is explicitly marked as an external contract with named downstream dependents ([call-webhook-jobs.job.ts#L34-L37](../../../packages/twenty-server/src/engine/metadata-modules/webhook/jobs/call-webhook-jobs.job.ts#L34-L37)) — a comment that changes reviewer behavior. Contracts deserve that visibility.

## Boundary leaks and tensions (judgment practice)

- **The bypass flag.** `shouldBypassPermissionChecks` threads through the ORM ([workspace-insert-query-builder.ts#L236](../../../packages/twenty-server/src/engine/twenty-orm/repository/workspace-insert-query-builder.ts#L236), [workspace-select-query-builder.ts#L370-L372](../../../packages/twenty-server/src/engine/twenty-orm/repository/workspace-select-query-builder.ts#L370-L372)). Necessary (system jobs, event payload reads), but every call site is a hole in the security boundary that reviews must treat as privileged. A boolean parameter as a security switch is cheaper than a separate privileged API but easier to misuse — defensible either way; know both sides.
- **Side effects inside the data layer.** Event emission happens *in the insert query builder* ([workspace-insert-query-builder.ts#L255-L275](../../../packages/twenty-server/src/engine/twenty-orm/repository/workspace-insert-query-builder.ts#L255-L275)). Win: no write path can forget to emit. Cost: the ORM layer now owns messaging concerns, and emission timing relative to transaction commit is decided implicitly (see [04-side-effects-async-and-reliability.md](04-side-effects-async-and-reliability.md)).
- **The base-class god-object gravity.** `CommonBaseQueryRunnerService` injects ~13 collaborators ([common-base-query-runner.service.ts#L64-L95](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-base-query-runner.service.ts#L64-L95)). Template-method keeps subclasses tiny, but every new cross-cutting concern lands in the base class — watch it grow. Alternative shape: a middleware/interceptor chain. Tradeoff: explicit pipeline order vs class inheritance rigidity.

## Front-side boundary

The record engine ([modules/object-record/](../../../packages/twenty-front/src/modules/object-record)) is generic; feature modules configure it with metadata. Leak to watch for as you read: any feature module reaching into record-engine internals (cache utils, table internals) instead of its public hooks — that coupling is how generic engines rot.

## Drill

Find where REST-specific input parsing ends and shared pipeline begins for `POST /rest/companies` (start at [rest-api-core.controller.ts](../../../packages/twenty-server/src/engine/api/rest/core/controllers/rest-api-core.controller.ts) → [rest-api-create-one.handler.ts](../../../packages/twenty-server/src/engine/api/rest/core/handlers/rest-api-create-one.handler.ts)). Self-grade — Basic: you find the handoff file. Solid: you name what shape converts at the boundary. Strong: you state one rule for what should *never* appear on the REST side (answer: anything permission- or validation-shaped).

## Interview angle

"Describe layered architecture in a system you know" — draw the stack above, then spend your time on the two *judgment* items: authorization on the shared path, and the bypass flag as a managed hole. That's what separates a mid answer from a memorized one. Cross-link: [08-interview-prep/04](../08-interview-prep/04-system-design-from-this-repo.md).
