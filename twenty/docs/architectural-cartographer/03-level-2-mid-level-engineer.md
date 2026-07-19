# 03 - Level 2: Mid-Level Engineer Guide

## 2.1 Full-Stack Architecture Diagram

```text
Browser
  |
  | React mount
  v
packages/twenty-front/src/index.tsx:10
  |
  v
App shell
packages/twenty-front/src/modules/app/components/App.tsx:20
  |
  | JotaiProvider, error boundary, i18n, icons, helmet
  v
Router shell
packages/twenty-front/src/modules/app/components/AppRouter.tsx:17
  |
  v
Route/provider runtime
packages/twenty-front/src/modules/app/components/AppRouterProviders.tsx:44
  |
  | ApolloProvider, metadata loader, auth provider, workspace provider, SSE, command menu
  v
Pages
packages/twenty-front/src/modules/app/hooks/useCreateAppRouter.tsx:121
  |
  | Example: RecordIndexPage route
  v
Feature modules
packages/twenty-front/src/modules/object-record
packages/twenty-front/src/modules/views
packages/twenty-front/src/modules/workspace
packages/twenty-front/src/modules/settings
  |
  | Apollo Client and Jotai state
  v
API transport
packages/twenty-front/src/modules/apollo/services/apollo.factory.ts:343
  |
  | /graphql, /metadata, /rest, file upload, streaming REST
  v
NestJS app
packages/twenty-server/src/main.ts:30
  |
  v
AppModule
packages/twenty-server/src/app.module.ts:49
  |
  | GraphQLModule, REST, MCP, ORM, CoreEngineModule, ModulesModule
  v
API layer
GraphQL: packages/twenty-server/src/app.module.ts:52
REST: packages/twenty-server/src/engine/api/rest/core/controllers/rest-api-core.controller.ts:24
Metadata REST: packages/twenty-server/src/engine/metadata-modules/view-sort/controllers/view-sort.controller.ts:38
Metadata GraphQL: packages/twenty-server/src/engine/metadata-modules/webhook/webhook.resolver.ts:24
  |
  | guards, middleware, decorators
  v
Service/business layer
packages/twenty-server/src/engine/api/rest/core/services/rest-api-core.service.ts:24
packages/twenty-server/src/engine/metadata-modules/view-sort/services/view-sort.service.ts:24
packages/twenty-server/src/engine/metadata-modules/webhook/webhook.service.ts:24
  |
  | TypeORM repositories, flat entity maps, workspace migrations
  v
Data layer
TypeORM core: packages/twenty-server/src/database/typeorm/typeorm.module.ts:9
Global workspace data source: packages/twenty-server/src/engine/twenty-orm/global-workspace-datasource/global-workspace-datasource.service.ts:16
Workspace migration runner: packages/twenty-server/src/engine/workspace-manager/workspace-migration
```

Type boundaries:

- Frontend generated GraphQL metadata types live in `packages/twenty-front/src/generated-metadata/graphql.ts:1`.
- Shared app/domain types live in `packages/twenty-shared/src/types`.
- Backend DTOs such as `ViewSortDTO` live in `packages/twenty-server/src/engine/metadata-modules/view-sort/dtos/view-sort.dto.ts`.
- Backend request context is represented by types such as `AuthenticatedRequest` imported at `packages/twenty-server/src/engine/api/rest/core/controllers/rest-api-core.controller.ts:19`.

## 2.2 TypeScript Type System Deep Dive

### Shared And Generated Types

The frontend uses generated GraphQL types from `packages/twenty-front/src/generated-metadata/graphql.ts:1`. The generated scalar map shows strong primitives for `ID`, `String`, `Boolean`, `Int`, `Float`, and `UUID`, but `ConnectionCursor`, `JSON`, `JSONObject`, and `Upload` are `any` at `packages/twenty-front/src/generated-metadata/graphql.ts:14`. That is a real weak spot: JSON fields are flexible by design, but every `any` is a place where runtime validation matters more.

Shared cross-package domain enums live in `twenty-shared`; for example, `ViewSortDirection` is defined in `packages/twenty-shared/src/types/ViewSortDirection.ts:1` and also appears in generated metadata output at `packages/twenty-front/src/generated-metadata/graphql.ts:5782`. A reviewer should check that frontend UI values, GraphQL schema values, and backend DTO values stay aligned.

### API Request And Response Shapes

GraphQL operations are typed through generated documents. `useLoadCurrentUser` calls `GetCurrentUserDocument` at `packages/twenty-front/src/modules/users/hooks/useLoadCurrentUser.ts:48`, and the generated fragment shape appears in `packages/twenty-front/src/generated-metadata/graphql.ts:7488`.

REST metadata controllers type their inputs and outputs explicitly. `ViewSortController.create` accepts `CreateViewSortInput` and returns `Promise<ViewSortDTO>` at `packages/twenty-server/src/engine/metadata-modules/view-sort/controllers/view-sort.controller.ts:89`. `WebhookController.update` accepts a local REST body shape and converts it to `UpdateWebhookInput` at `packages/twenty-server/src/engine/metadata-modules/webhook/controllers/webhook.controller.ts:65`.

Dynamic object REST APIs are less statically obvious. `RestApiCoreController` accepts `AuthenticatedRequest` for wildcard routes at `packages/twenty-server/src/engine/api/rest/core/controllers/rest-api-core.controller.ts:33`, and `RestApiCoreService` dispatches by parsed path/id at `packages/twenty-server/src/engine/api/rest/core/services/rest-api-core.service.ts:56`. This is type-safe at the request-context level but runtime-driven at the object-schema level.

### Useful Type Protection

- `CurrentUser` uses `Pick<User, ...>` at `packages/twenty-front/src/modules/auth/states/currentUserState.ts:4`, so global state cannot pretend to contain the full generated user.
- `createAtomState<ValueType>` at `packages/twenty-front/src/modules/ui/utilities/state/jotai/utils/createAtomState.ts:27` makes persisted and in-memory atoms preserve their value type.
- `Options` for `ApolloFactory` at `packages/twenty-front/src/modules/apollo/services/apollo.factory.ts:49` makes auth, workspace, callbacks, cache, and URI requirements explicit.
- `ViewSortController.update` constrains update body to `{ direction?: ViewSortDirection }` at `packages/twenty-server/src/engine/metadata-modules/view-sort/controllers/view-sort.controller.ts:103`.

### Where Types Fall Short

- Generated metadata scalars use `any` for JSON-like fields at `packages/twenty-front/src/generated-metadata/graphql.ts:16`.
- `WebhookController.update` uses a manually declared object body at `packages/twenty-server/src/engine/metadata-modules/webhook/controllers/webhook.controller.ts:69`; the GraphQL resolver uses `UpdateWebhookInput` directly at `packages/twenty-server/src/engine/metadata-modules/webhook/webhook.resolver.ts:57`.
- Dynamic REST object payloads route through `AuthenticatedRequest` at `packages/twenty-server/src/engine/api/rest/core/controllers/rest-api-core.controller.ts:33`, so object-field validation must happen in parsers/handlers rather than controller signatures.

## 2.3 State Management Deep Dive: Current User State

Current user lifecycle:

1. State definition: `CurrentUser` is defined as a `Pick<User, ...>` at `packages/twenty-front/src/modules/auth/states/currentUserState.ts:4`.
2. Atom creation: `currentUserState` is created with default `null` at `packages/twenty-front/src/modules/auth/states/currentUserState.ts:18`.
3. Store creation: the app-level Jotai store is created in `packages/twenty-front/src/modules/ui/utilities/state/jotai/jotaiStore.ts:4`.
4. Store provision: `App` passes that store to `JotaiProvider` at `packages/twenty-front/src/modules/app/components/App.tsx:20`.
5. User load: `useLoadCurrentUser` queries `GetCurrentUserDocument` at `packages/twenty-front/src/modules/users/hooks/useLoadCurrentUser.ts:48`.
6. State update: `setCurrentUser(user)` runs at `packages/twenty-front/src/modules/users/hooks/useLoadCurrentUser.ts:65`.
7. Related state fan-out: workspace members, workspace, permissions, locale, and auth bypass providers update at `packages/twenty-front/src/modules/users/hooks/useLoadCurrentUser.ts:67`.
8. Consuming component: `AppRouter` reads `currentUserState` at `packages/twenty-front/src/modules/app/components/AppRouter.tsx:10`.
9. Behavior from state: admin page availability is derived at `packages/twenty-front/src/modules/app/components/AppRouter.tsx:12`.
10. Error path: if the query has an error, `useLoadCurrentUser` throws at `packages/twenty-front/src/modules/users/hooks/useLoadCurrentUser.ts:53`; Apollo unauthenticated handling also clears auth/workspace atoms at `packages/twenty-front/src/modules/apollo/hooks/useApolloFactory.ts:68`.

State flow:

```text
Apollo query GetCurrentUserDocument
  -> useLoadCurrentUser
  -> currentUserState
  -> AppRouter
  -> useCreateAppRouter(admin enabled?)
  -> Settings/Admin route availability
```

Senior review instinct: when changing current user shape, inspect `packages/twenty-front/src/modules/users/graphql/fragments/userQueryFragment.ts`, `packages/twenty-front/src/modules/auth/states/currentUserState.ts`, `packages/twenty-front/src/modules/users/hooks/useLoadCurrentUser.ts`, and all `currentUserState` readers.

## 2.4 API Contract

### Runtime API Families

| API family | Endpoint shape | Code | Auth/authorization | Contract notes |
|---|---|---|---|---|
| Core GraphQL | `/graphql` | Bootstrapped by `GraphQLModule.forRootAsync` at `packages/twenty-server/src/app.module.ts:52` | Request hydrated by `GraphQLHydrateRequestFromTokenMiddleware` and `WorkspaceAuthContextMiddleware` at `packages/twenty-server/src/app.module.ts:116` | Primary generated app API. |
| Metadata GraphQL | `/metadata` | Same GraphQL module plus metadata API modules at `packages/twenty-server/src/app.module.ts:68` | Same hydration/context middleware at `packages/twenty-server/src/app.module.ts:122` | Used by frontend `ApolloProvider` at `packages/twenty-front/src/modules/apollo/components/ApolloProvider.tsx:13`. |
| Admin GraphQL | `/admin-panel` | Admin panel module imported at `packages/twenty-server/src/app.module.ts:69` | Hydrated by middleware at `packages/twenty-server/src/app.module.ts:129` | Admin access is UI-gated by `AppRouter.tsx:12`, but backend guards remain decisive. |
| Dynamic REST object API | `/rest/*path` | `RestApiCoreController` at `packages/twenty-server/src/engine/api/rest/core/controllers/rest-api-core.controller.ts:24` | `JwtAuthGuard`, `WorkspaceAuthGuard`, `CustomPermissionGuard` at controller line 25 | Handles create, read, update, delete, restore, merge, groupBy, duplicates. |
| Metadata REST view sorts | `/rest/metadata/viewSorts` | `ViewSortController` at `packages/twenty-server/src/engine/metadata-modules/view-sort/controllers/view-sort.controller.ts:38` | `WorkspaceAuthGuard` plus method permission guards | Example of metadata CRUD via REST. |
| Metadata REST webhooks | `/rest/webhooks`, `/rest/metadata/webhooks` | `WebhookController` at `packages/twenty-server/src/engine/metadata-modules/webhook/controllers/webhook.controller.ts:28` | JWT, workspace auth, API keys/webhooks settings permission at line 29 | Same service also exposed through metadata GraphQL resolver. |
| MCP | `/mcp` | `McpModule` imported at `packages/twenty-server/src/app.module.ts:71` and method guard middleware at `packages/twenty-server/src/app.module.ts:136` | `McpMethodGuardMiddleware` plus controller guards | Tool/API surface for MCP clients. |

RESTfulness rating: 3/5. Metadata controllers such as `ViewSortController` are conventional REST, but dynamic `rest/*path` trades strict resource explicitness for metadata-driven object flexibility.

Type safety rating: 3/5. GraphQL generated types are strong, but `any` JSON scalars at `packages/twenty-front/src/generated-metadata/graphql.ts:16` and wildcard REST payloads reduce compile-time coverage.

Error consistency rating: 4/5. Controllers consistently use filters such as `RestApiExceptionFilter` at `packages/twenty-server/src/engine/api/rest/core/controllers/rest-api-core.controller.ts:26` and specialized metadata filters at `packages/twenty-server/src/engine/metadata-modules/view-sort/controllers/view-sort.controller.ts:40`.

## 2.5 Component Interaction Map

| Component A | Component B | Communication method |
|---|---|---|
| `App` | `AppRouter` | Provider context and direct render at `packages/twenty-front/src/modules/app/components/App.tsx:36`. |
| `AppRouter` | `useCreateAppRouter` | Function call with derived flags at `packages/twenty-front/src/modules/app/components/AppRouter.tsx:18`. |
| `AppRouterProviders` | nested pages | React Router `<Outlet />` at `packages/twenty-front/src/modules/app/components/AppRouterProviders.tsx:72`. |
| `AppRouterProviders` | Apollo consumers | Apollo context from `ApolloProvider` at `packages/twenty-front/src/modules/app/components/AppRouterProviders.tsx:44`. |
| `useLoadCurrentUser` | `AppRouter` | Shared Jotai atom `currentUserState`, written at `useLoadCurrentUser.ts:65` and read at `AppRouter.tsx:10`. |
| `ApolloProvider` | `ApolloFactory` | Hook-based construction through `useApolloFactory` at `packages/twenty-front/src/modules/apollo/components/ApolloProvider.tsx:13`. |
| `ApolloFactory` | backend APIs | Apollo links add auth/workspace/app-version headers at `packages/twenty-front/src/modules/apollo/services/apollo.factory.ts:111`. |

The important path is not prop drilling; it is provider composition plus atom-backed state. The app renders pages only after global runtime capabilities are in place. That choice makes feature pages thinner, but it raises the cost of understanding boot behavior because many side effects mount above `<Outlet />`.

## 2.6 Full-Stack Feature Trace: Creating A View Sort

1. UI event handler: a view/table sort interaction eventually persists through the `views` and `object-record` modules. Start searches from `packages/twenty-front/src/modules/views/hooks/internal/usePerformViewFilterGroupAPIPersist.ts` and neighboring view-sort persist hooks because view state is organized under `modules/views`.
2. State/data trigger: Apollo client comes from `AppRouterProviders` through `ApolloProvider` at `packages/twenty-front/src/modules/app/components/AppRouterProviders.tsx:44`.
3. HTTP request: Apollo link chain chooses GraphQL/REST, adds auth headers, and sends to server; auth headers are built at `packages/twenty-front/src/modules/apollo/services/apollo.factory.ts:111`.
4. Backend route: REST metadata creation enters `ViewSortController.create` at `packages/twenty-server/src/engine/metadata-modules/view-sort/controllers/view-sort.controller.ts:87`.
5. Middleware chain: REST requests run `RestCoreMiddleware` and `WorkspaceAuthContextMiddleware` at `packages/twenty-server/src/app.module.ts:140`; the controller also uses `WorkspaceAuthGuard` at `view-sort.controller.ts:39`.
6. Permission: create is guarded by `CreateViewSortPermissionGuard` at `packages/twenty-server/src/engine/metadata-modules/view-sort/controllers/view-sort.controller.ts:88`.
7. Service: controller delegates to `viewSortService.createOne` at `packages/twenty-server/src/engine/metadata-modules/view-sort/controllers/view-sort.controller.ts:93`.
8. Metadata read: service loads application and flat maps at `packages/twenty-server/src/engine/metadata-modules/view-sort/services/view-sort.service.ts:41`.
9. Migration: service validates/builds/runs a workspace migration at `packages/twenty-server/src/engine/metadata-modules/view-sort/services/view-sort.service.ts:63`.
10. Response: service recomputes maps and converts flat metadata back to DTO at `packages/twenty-server/src/engine/metadata-modules/view-sort/services/view-sort.service.ts:87`.

What breaks if a step changes incorrectly: bad frontend headers cause 401s; skipped guards allow unauthorized sort changes; skipped migration validation can corrupt metadata; stale flat maps return a DTO that does not match persisted workspace state.

## 2.7 Diff Reading Exercise

Hypothetical change: add `description` to view sorts so users can annotate why a saved sort exists.

Review order:

1. Read DTO/input changes first: `packages/twenty-server/src/engine/metadata-modules/view-sort/dtos`.
2. Read flat conversion utilities next: `packages/twenty-server/src/engine/metadata-modules/flat-view-sort/utils`.
3. Read service migration path: `packages/twenty-server/src/engine/metadata-modules/view-sort/services/view-sort.service.ts:34`.
4. Read REST and GraphQL controllers/resolvers: `packages/twenty-server/src/engine/metadata-modules/view-sort/controllers/view-sort.controller.ts:87`.
5. Read frontend generated metadata types after codegen: `packages/twenty-front/src/generated-metadata/graphql.ts`.
6. Read UI persist hook and tests under `packages/twenty-front/src/modules/views`.

TypeScript implications: generated GraphQL types must include the field, DTOs must expose it, REST body types must accept it, and frontend fragment/query code must request it. Assumptions: existing migrations can add the metadata field safely; old records without description must render as `null` or empty. Edge cases: unauthorized users updating descriptions, blank vs null description, stale generated types, migration failure. Senior review questions: is this metadata or user data, does it belong in flat maps, how does it affect export/import, and what happens to existing workspaces?

Quiz:

1. Which service method would run when a new view sort is created?
2. Why is codegen required after GraphQL schema changes?
3. Which guard protects view-sort creation?
4. Where would stale metadata cache show up?
5. What test would catch a missing generated field?

## 2.8 Non-Obvious Patterns

### Pattern: Apollo Handles GraphQL And REST

```ts
// Creates upload GraphQL transport.
const uploadLink = new UploadHttpLink({ uri });

// Creates streaming REST transport.
const streamingRestLink = new StreamingRestLink({ uri: REST_API_BASE_URL });

// Creates normal REST transport.
const restLink = new RestLink({ uri: REST_API_BASE_URL });

// All links become one Apollo chain.
const links = [errorLink, authLink, ...(extraLinks || []), retryLink, streamingRestLink, restLink, uploadLink] as ApolloLink[];
```

Implemented at `packages/twenty-front/src/modules/apollo/services/apollo.factory.ts:98` and `packages/twenty-front/src/modules/apollo/services/apollo.factory.ts:343`. The likely reason is one transport abstraction for GraphQL, REST, upload, auth, retries, and errors. Tradeoff: powerful central control, but debugging network behavior requires understanding Apollo links.

### Pattern: Metadata Changes Flow Through Workspace Migrations

```ts
const validateAndBuildResult =
  await this.workspaceMigrationValidateBuildAndRunService.validateBuildAndRunWorkspaceMigration({
    allFlatEntityOperationByMetadataName: {
      viewSort: {
        flatEntityToCreate: [],
        flatEntityToDelete: [],
        flatEntityToUpdate: [optimisticallyUpdatedFlatViewSort],
      },
    },
    workspaceId,
    isSystemBuild: false,
    applicationUniversalIdentifier: workspaceCustomFlatApplication.universalIdentifier,
  });
```

Implemented in `packages/twenty-server/src/engine/metadata-modules/view-sort/services/view-sort.service.ts:131`. This avoids treating metadata as plain CRUD. Tradeoff: safer consistency, more indirection.

### Pattern: Current User Query Fans Out Into Many Client States

```ts
setCurrentUser(user);
setCurrentWorkspaceMembers(user.workspaceMembers);
setAvailableWorkspaces(user.availableWorkspaces);
setCurrentUserWorkspace({ permissionFlags: user.currentUserWorkspace.permissionFlags ?? [] });
setCurrentWorkspace(workspace);
```

Implemented across `packages/twenty-front/src/modules/users/hooks/useLoadCurrentUser.ts:65`. This gives components small focused atoms, but every auth-state change must keep the fan-out consistent.

## 2.9 Mid-Level Socratic Checkpoint

| Question | Strong answer should include |
|---|---|
| Why does current user state live in Jotai instead of only Apollo cache? | Multiple non-query consumers need lightweight global auth/workspace state; see writes in `useLoadCurrentUser.ts:65` and reads in `AppRouter.tsx:10`. |
| How does a metadata view sort creation flow end to end? | UI -> Apollo -> metadata REST/GraphQL -> guards -> `ViewSortService.createOne` -> flat maps -> workspace migration -> DTO response. |
| What does `Pick<User, ...>` protect against? | Over-storing or relying on unqueried generated user fields; see `currentUserState.ts:4`. |
| What happens if `CreateViewSortPermissionGuard` is removed? | Users with workspace access could create view sorts without the intended permission check at `view-sort.controller.ts:88`. |
| What abstraction is `ApolloFactory` hiding? | Auth headers, token renewal, app version mismatch, retry, REST links, upload links, Sentry handling. |
| What is fragile? | Metadata flows are powerful but complex: generated types, flat maps, migration validation, cache recompute all need alignment. |
| How would you add a new React route? | Add page under `src/pages`, lazy import in `useCreateAppRouter.tsx`, route path from `AppPath`, ensure providers/state needs are satisfied. |
| What would you focus on reviewing a PR touching `RestApiCoreController`? | Guards, path parsing, one-vs-many dispatch, soft delete semantics, response codes, exception filter consistency. |

