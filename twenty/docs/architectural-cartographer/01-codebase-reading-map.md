# 01 - Codebase Reading Map

## Mental Model In One Paragraph

Twenty thinks like a metadata-driven CRM workspace, not like a fixed CRUD app. The frontend starts as a provider shell that loads config, auth, workspace metadata, theme, Apollo clients, SSE, command menu, and layout before rendering feature routes; see `packages/twenty-front/src/index.tsx:10`, `packages/twenty-front/src/modules/app/components/App.tsx:20`, and `packages/twenty-front/src/modules/app/components/AppRouterProviders.tsx:44`. The backend starts as a NestJS engine that installs GraphQL, REST, MCP, workspace auth context, global workspace data sources, and metadata modules before business code handles records, views, webhooks, workflow, and workspace state; see `packages/twenty-server/src/main.ts:30`, `packages/twenty-server/src/app.module.ts:52`, and `packages/twenty-server/src/app.module.ts:114`. The key senior-level idea is that a "record" or "view" feature is usually not owned by one screen and one endpoint; it flows through workspace metadata, generated GraphQL types, Apollo/Jotai state, Nest guards, migration services, and TypeORM-backed metadata entities.

## Reading Order

| Order | File | Why now | Understand first | Explain after reading |
|---:|---|---|---|---|
| 1 | `package.json` | It names the repo as a Yarn/Nx workspace and shows the root `start` command. | Workspaces collect many packages under one repo. | Why `twenty-front`, `twenty-server`, and the worker start together through `package.json:44`. |
| 2 | `nx.json` | Nx controls how projects build, lint, and depend on each other. | Monorepos need task orchestration. | Why package boundaries matter before reading app code. |
| 3 | `packages/twenty-front/project.json` | It identifies the frontend as an app and shows Vite, lint, test, and Storybook targets. | React apps have dev server and build targets. | How the frontend starts through `packages/twenty-front/project.json:37` and checks code through `packages/twenty-front/project.json:66`. |
| 4 | `packages/twenty-front/src/index.tsx` | This is the browser-to-React handoff. | React roots render one top component. | Why `ReactDOM.createRoot` at `packages/twenty-front/src/index.tsx:10` hands control to `<App />` at `packages/twenty-front/src/index.tsx:14`. |
| 5 | `packages/twenty-front/src/modules/app/components/App.tsx` | This shows the first application-owned provider stack. | Providers make global capabilities available below them. | Why Jotai, error boundaries, i18n, icons, and router are composed before any page renders at `packages/twenty-front/src/modules/app/components/App.tsx:20`. |
| 6 | `packages/twenty-front/src/modules/app/components/AppRouterProviders.tsx` | This is the real runtime shell around every route. | Route outlets render nested pages. | Why Apollo, config, metadata, auth, SSE, snackbars, command menu, and effects sit above `<Outlet />` at `packages/twenty-front/src/modules/app/components/AppRouterProviders.tsx:72`. |
| 7 | `packages/twenty-front/src/modules/app/hooks/useCreateAppRouter.tsx` | This maps URL paths to pages. | React Router turns location into components. | Why record pages, onboarding pages, auth pages, settings, and page layouts are lazy-loaded at `packages/twenty-front/src/modules/app/hooks/useCreateAppRouter.tsx:19`. |
| 8 | `packages/twenty-server/project.json` | It shows how the backend runs, tests, migrates, and starts workers. | Nest apps often have a web process and worker process. | Why server start is `packages/twenty-server/project.json:51` and worker start is `packages/twenty-server/project.json:118`. |
| 9 | `packages/twenty-server/src/main.ts` | This is the Node/Nest process entry point. | Nest creates an application from a root module. | How middleware, session storage, upload handling, config, logging, and `app.listen` are installed at `packages/twenty-server/src/main.ts:30`. |
| 10 | `packages/twenty-server/src/app.module.ts` | This is the backend module map. | Nest modules group controllers, providers, and imports. | Why GraphQL, REST, MCP, ORM, core modules, metadata modules, and middleware are attached in `packages/twenty-server/src/app.module.ts:49`. |

## Core Data Flows

1. Authenticated Workspace Bootstrap Flow: browser renders `App` at `packages/twenty-front/src/index.tsx:14`, providers mount in `AppRouterProviders` at `packages/twenty-front/src/modules/app/components/AppRouterProviders.tsx:44`, Apollo sends auth headers in `packages/twenty-front/src/modules/apollo/services/apollo.factory.ts:111`, and backend workspace context middleware runs in `packages/twenty-server/src/app.module.ts:114`.

2. Metadata-Driven Record Flow: React routes to record pages through `packages/twenty-front/src/modules/app/hooks/useCreateAppRouter.tsx:214`, object-record utilities generate dynamic queries such as `packages/twenty-front/src/modules/object-record/utils/generateFindManyRecordsQuery.ts`, Apollo reaches `/graphql` or `/metadata` through `packages/twenty-front/src/modules/apollo/hooks/useApolloFactory.ts:45`, and the backend routes dynamic REST object access through `packages/twenty-server/src/engine/api/rest/core/controllers/rest-api-core.controller.ts:24`.

3. Metadata Mutation -> Workspace Migration Flow: a UI mutation such as view sort or webhook update calls metadata GraphQL/REST; controllers like `packages/twenty-server/src/engine/metadata-modules/view-sort/controllers/view-sort.controller.ts:87` delegate to services like `packages/twenty-server/src/engine/metadata-modules/view-sort/services/view-sort.service.ts:34`, which validate and run workspace migrations through `validateBuildAndRunWorkspaceMigration` at `packages/twenty-server/src/engine/metadata-modules/view-sort/services/view-sort.service.ts:63`.

## Pre-Reading Checklist

By the end of this guide, you should be able to answer:

1. Why does the frontend need both `App.tsx` and `AppRouterProviders.tsx`?
2. Why does `useApolloFactory` know about current workspace, current workspace member, token renewal, and app version?
3. Why is `currentUserState` a `Pick<User, ...> | null` instead of a plain object?
4. How does a route such as `AppPath.RecordIndexPage` become a lazily loaded page?
5. How does the backend attach workspace context to GraphQL and REST requests?
6. Why does the backend expose wildcard REST routes under `rest/*path`?
7. What is the difference between core object records and metadata objects such as view sorts and webhooks?
8. Where would you look first if a user is signed out unexpectedly after a GraphQL request?
9. Where would you look first if a view sort saves in the UI but does not persist after refresh?
10. What files should a reviewer inspect when a PR changes an API response shape?

## Red Flags Checklist

Watch for these while reading and reviewing:

1. Auth state changes that update some Jotai atoms but not the full set updated by `useLoadCurrentUser` at `packages/twenty-front/src/modules/users/hooks/useLoadCurrentUser.ts:65`.
2. Apollo error handling that bypasses token renewal paths in `packages/twenty-front/src/modules/apollo/services/apollo.factory.ts:270`.
3. Dynamic REST route changes that ignore `RestApiCoreService` one-vs-many dispatch in `packages/twenty-server/src/engine/api/rest/core/services/rest-api-core.service.ts:56`.
4. Metadata mutations that skip workspace migration validation used in `packages/twenty-server/src/engine/metadata-modules/view-sort/services/view-sort.service.ts:63`.
5. Unsafe generated scalar assumptions where `JSON`, `JSONObject`, or `Upload` are typed as `any` in `packages/twenty-front/src/generated-metadata/graphql.ts:16`.
6. Permission guard removal around metadata endpoints such as `SettingsPermissionGuard` in `packages/twenty-server/src/engine/metadata-modules/webhook/controllers/webhook.controller.ts:29`.
7. Component changes that assume metadata is loaded before `MinimalMetadataGater` allows rendering at `packages/twenty-front/src/modules/app/components/AppRouterProviders.tsx:53`.
8. File upload changes that exceed max size assumptions configured in `packages/twenty-server/src/main.ts:66` and `packages/twenty-server/src/main.ts:74`.
9. Code that mutates workspace metadata directly instead of using flat-entity maps and migration services, as seen in `packages/twenty-server/src/engine/metadata-modules/webhook/webhook.service.ts:117`.
10. Tests that mock Apollo without matching the real headers, retry, and token renewal behavior from `packages/twenty-front/src/modules/apollo/services/apollo.factory.ts:111`.

