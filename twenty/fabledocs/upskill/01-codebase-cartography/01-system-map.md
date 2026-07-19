# System Map

## Repo shape: Nx monorepo, Yarn 4 workspaces

Workspaces are declared in [package.json](../../../package.json#L46-L60). The ones that matter daily:

```
packages/
├── twenty-server/    # NestJS API: GraphQL (/graphql, /metadata), REST, workers, cron
├── twenty-front/     # React 18 SPA (Vite): the CRM UI
├── twenty-shared/    # Types + utils shared front/back (build this first)
├── twenty-ui/        # Design-system components (no product logic)
├── twenty-emails/    # React Email templates
├── twenty-e2e-testing/ # Playwright end-to-end suite
├── twenty-zapier/    # Zapier integration (consumes the webhook contract)
├── twenty-docker/    # docker-compose for dev/prod
└── twenty-cli / twenty-sdk / twenty-apps ... # extension platform
```

## Runtime surfaces

```
Browser (React SPA, port 3001)
   │  GraphQL over HTTP: /graphql (records), /metadata (schema-of-the-schema)
   ▼
twenty-server (NestJS, port 3000)  ── REST: /rest/*  ── MCP endpoint ── OpenAPI
   │            │
   │            └── emits in-process events ──► enqueues BullMQ jobs (Redis)
   ▼                                                     │
PostgreSQL (core schema + per-workspace schemas)         ▼
Redis (cache, sessions, queues)                 queue-worker process
ClickHouse (optional analytics)                 (webhooks, workflows, sync jobs)
```

Three processes run in dev via the root `start` script ([package.json](../../../package.json#L44)): server, front, and worker. The worker is a *separate process* consuming the same codebase's `@Processor` classes — e.g. [call-webhook.job.ts](../../../packages/twenty-server/src/engine/metadata-modules/webhook/jobs/call-webhook.job.ts#L15-L21). If you forget the worker is separate, you'll wonder why your webhook code "never runs" while debugging the server process.

## Ownership map (server)

| Area | Directory | Owns |
| --- | --- | --- |
| API layer | [engine/api/](../../../packages/twenty-server/src/engine/api) | GraphQL schema generation, resolvers, REST handlers, MCP; **no business rules** |
| Shared query pipeline | [engine/api/common/](../../../packages/twenty-server/src/engine/api/common) | The runner pipeline both GraphQL and REST delegate to |
| Core modules | [engine/core-modules/](../../../packages/twenty-server/src/engine/core-modules) | Auth, billing, files, feature flags, message queue, metrics — workspace-agnostic platform services |
| Metadata modules | [engine/metadata-modules/](../../../packages/twenty-server/src/engine/metadata-modules) | Object/field metadata, permissions, roles, webhooks, views — the "schema of the schema" |
| Tenant ORM | [engine/twenty-orm/](../../../packages/twenty-server/src/engine/twenty-orm) | Custom TypeORM extension: per-workspace datasources, permission-enforcing query builders |
| Workspace lifecycle | [engine/workspace-manager/](../../../packages/twenty-server/src/engine/workspace-manager) | Workspace creation, metadata sync, workspace migrations |
| Product features | [modules/](../../../packages/twenty-server/src/modules) | CRM domain: company, person, opportunity, messaging sync, calendar, workflow |

The split to internalize: `engine/` is the *platform* (could run any metadata-defined app); `modules/` is the *product* (a CRM defined on top of that platform). That is a real architectural boundary, and it's the best single sentence you can say about Twenty in an interview.

## Ownership map (front)

| Area | Directory | Owns |
| --- | --- | --- |
| Record engine | [modules/object-record/](../../../packages/twenty-front/src/modules/object-record) | Generic record CRUD, tables, boards, cells, cache — works for *any* object |
| Metadata client | [modules/object-metadata/](../../../packages/twenty-front/src/modules/object-metadata) | Fetching/using object definitions |
| Feature UIs | `modules/companies`, `modules/people`, `modules/settings`, `modules/workflow`… | Thin: mostly configuration of the generic record engine |
| State/UI kit | `modules/ui/`, Jotai atoms per module (`states/`) | Cross-cutting UI utilities |
| Apollo layer | [modules/apollo/](../../../packages/twenty-front/src/modules/apollo) | Clients + optimistic-effect machinery |

## Public interfaces vs private internals

**Public (contract — changing these has blast radius outside this repo):** the generated GraphQL schemas on `/graphql` and `/metadata`; the REST API under [engine/api/rest/core/](../../../packages/twenty-server/src/engine/api/rest/core); the webhook payload shape (the comment at [call-webhook-jobs.job.ts](../../../packages/twenty-server/src/engine/metadata-modules/webhook/jobs/call-webhook-jobs.job.ts#L34-L37) explicitly warns that Zapier and the OpenAPI schema depend on it); the SDK/CLI extension surface.

**Private (refactor freely with tests):** the runner pipeline internals, twenty-orm query builders, Jotai atom shapes, cache utils.

Drill: pick `engine/core-modules/record-position` and `modules/opportunity` and decide which side of the platform/product boundary each sits on and why. Self-grade — Basic: correct placement. Solid: justify via what each imports. Strong: name one thing that would force `record-position` to move into `modules/` and why that would be a design smell.
