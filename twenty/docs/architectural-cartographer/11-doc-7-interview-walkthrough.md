# 11 - Doc 7: Interview Walkthrough

## 60-Second Summary: Non-Technical

Twenty is an open-source CRM platform for managing workspace data such as people, companies, tasks, workflows, settings, webhooks, and integrations. What makes it interesting is that much of the app is driven by workspace metadata, so teams can shape records, views, permissions, and layouts rather than relying only on hard-coded screens. The frontend and backend cooperate to load a user's workspace context, render the correct CRM pages, and enforce permissions on every API path.

Concrete anchor: the frontend starts at `packages/twenty-front/src/index.tsx:14`, and the backend starts at `packages/twenty-server/src/main.ts:30`.

## 60-Second Summary: Technical

Twenty is an Nx monorepo with a React/Vite frontend and a NestJS backend. The frontend composes global state and runtime providers through `App.tsx` and `AppRouterProviders.tsx`, uses Jotai for client state, and Apollo for GraphQL, REST, upload, retry, and auth transport. The backend composes GraphQL, REST, MCP, TypeORM, core engine modules, business modules, workspace middleware, guards, and metadata migration services through `AppModule`. A lot of product behavior is metadata-driven: object records, view sorts, webhooks, and workspace configuration flow through generated GraphQL types, flat entity maps, permission guards, and workspace migration validation.

Concrete anchors: `packages/twenty-front/src/modules/app/components/AppRouterProviders.tsx:44`, `packages/twenty-front/src/modules/apollo/services/apollo.factory.ts:343`, and `packages/twenty-server/src/app.module.ts:49`.

## Architecture Explanation For A Senior Engineer

The browser entry is minimal: `ReactDOM.createRoot` at `packages/twenty-front/src/index.tsx:10` renders `<App />` at line 14. `App` provides Jotai, error handling, i18n, icons, Helmet, click-outside context, and then hands off to `AppRouter` at `packages/twenty-front/src/modules/app/components/App.tsx:36`. `AppRouter` reads `currentUserState` at `packages/twenty-front/src/modules/app/components/AppRouter.tsx:10` to decide admin route availability.

The real application shell is `AppRouterProviders`, which installs Apollo, metadata loading, auth, workspace effects, SSE, command menu, snackbars, dialogs, page title, favicon, and route outlet at `packages/twenty-front/src/modules/app/components/AppRouterProviders.tsx:44`. API transport is centralized in `ApolloFactory`, which builds auth headers at `packages/twenty-front/src/modules/apollo/services/apollo.factory.ts:111`, token renewal at line 162, error handling at line 270, and link composition at line 343.

On the backend, `main.ts` creates a Nest Express application from `AppModule` at `packages/twenty-server/src/main.ts:30`, configures sessions at line 56, body parsers at line 66, GraphQL upload middleware at line 74, and listens at line 93. `AppModule` imports GraphQL, ORM, core engine, business modules, REST, MCP, metadata modules, and middleware at `packages/twenty-server/src/app.module.ts:49`. Middleware attaches token/workspace context to GraphQL, metadata, admin-panel, and REST routes at `packages/twenty-server/src/app.module.ts:114`.

## Feature Walkthrough: View Sort Creation

When a user saves a sort on a CRM view, the frontend sends an authenticated Apollo request using headers built in `packages/twenty-front/src/modules/apollo/services/apollo.factory.ts:111`. The backend REST metadata endpoint is `POST /rest/metadata/viewSorts`, implemented by `ViewSortController.create` at `packages/twenty-server/src/engine/metadata-modules/view-sort/controllers/view-sort.controller.ts:87`. The controller requires `CreateViewSortPermissionGuard` at line 88 and delegates to `ViewSortService.createOne` at line 93.

The service loads the workspace custom application at `packages/twenty-server/src/engine/metadata-modules/view-sort/services/view-sort.service.ts:41`, reads flat field/view maps at line 48, converts input into a flat entity at line 56, validates/builds/runs a workspace migration at line 63, recomputes flat view sort maps at line 87, and returns a `ViewSortDTO` at line 95. The important point is that this is not direct CRUD; it is metadata mutation through migration validation.

## Architectural Tradeoff To Discuss

Twenty centralizes API transport in `ApolloFactory`, including GraphQL, REST, streaming REST, uploads, auth headers, retries, Sentry reporting, token renewal, and app-version handling at `packages/twenty-front/src/modules/apollo/services/apollo.factory.ts:343`. The upside is consistent behavior across the app. The tradeoff is that debugging a single failed request requires understanding the link chain, not just the component that made the request.

A second tradeoff is dynamic metadata-driven APIs. `RestApiCoreController` can handle many object routes through wildcard paths at `packages/twenty-server/src/engine/api/rest/core/controllers/rest-api-core.controller.ts:24`. The upside is flexibility for custom CRM objects. The tradeoff is that contracts are less obvious from the controller signature alone.

## One Bug I Found And How I Would Debug It

Potential bug class: a user is signed out unexpectedly when multiple Apollo requests fail at the same time. I would start at `ApolloFactory` because it has a shared `renewalPromise` at `packages/twenty-front/src/modules/apollo/services/apollo.factory.ts:41`, token renewal logic at line 162, and unauthenticated handling at line 270. I would write a test that returns two concurrent `UNAUTHENTICATED` errors and asserts only one renewal request runs and auth state is not cleared unless renewal fails.

## One Improvement I Would Make Next

I would split `AppRouterProviders` into named provider groups without changing behavior. Today the provider stack starts at `packages/twenty-front/src/modules/app/components/AppRouterProviders.tsx:44` and mixes config, metadata, auth, Apollo, UI, SSE, command menu, support chat, and route effects. Grouping them into shells like `DataProviders`, `WorkspaceRuntimeProviders`, and `GlobalUIProviders` would make onboarding and review safer, as long as tests preserve order and side effects.

