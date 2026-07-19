# Mission 25: Write the Docs That Don't Exist

**Tier:** Senior  
**Time Estimate:** 120 minutes  
**Goal:** You will leave behind a compact reference suite that makes future onboarding, debugging, review, and ownership easier.

**The Concept:**

Good internal docs are not summaries. They are maps for future action: where to start, what to inspect, what risks matter, and how to explain the system under pressure.

**Design Intent (Before You Read the Code):**

1. Files involved: all anchor files from Missions 1-24, especially `index.tsx:1-14`, `App.tsx:1-46`, `AppRouterProviders.tsx:39-96`, `apollo.factory.ts:98-365`, `main.ts:1-96`, `app.module.ts:49-146`, `rest-api-core.controller.ts:24-163`, `dashboard.controller.ts:1-25`, `view-sort.service.ts:34-170`.
2. This mission owns the permanent artifacts that a learner returns to later.
3. Pattern: short, actionable, file-anchored docs.
4. Poor docs create confidence theater: they feel complete but do not help someone trace, review, debug, or explain.

**Find It In The Code:**

Use these anchors while reading the reference docs:

```text
Frontend boot: packages/twenty-front/src/index.tsx:1-14
App shell: packages/twenty-front/src/modules/app/components/App.tsx:1-46
Runtime providers: packages/twenty-front/src/modules/app/components/AppRouterProviders.tsx:39-96
Router: packages/twenty-front/src/modules/app/hooks/useCreateAppRouter.tsx:109-260
Current user load: packages/twenty-front/src/modules/users/hooks/useLoadCurrentUser.ts:24-153
Apollo transport: packages/twenty-front/src/modules/apollo/services/apollo.factory.ts:98-365
Backend boot: packages/twenty-server/src/main.ts:1-96
Backend module map: packages/twenty-server/src/app.module.ts:49-146
Simple route: packages/twenty-server/src/modules/dashboard/controllers/dashboard.controller.ts:1-25
Dynamic REST: packages/twenty-server/src/engine/api/rest/core/controllers/rest-api-core.controller.ts:24-163
Metadata service: packages/twenty-server/src/engine/metadata-modules/view-sort/services/view-sort.service.ts:34-170
```

**The "Aha" Moment:**

The best docs teach where to look next.

**Socratic Checkpoint:**

1. Which doc would help a brand-new contributor first?
2. Which doc would help during a production incident?
3. Which doc would help during PR review?
4. Which doc would help when adding a feature?
5. Which doc would help in an interview?

Self-grade rubric: strong answers map use case to artifact, not artifact to title.

**Connects To:**

- This mission closes the learning path and becomes the reference suite for future practice.

---

# Doc 1 - Junior Onboarding Guide

## Setup

The root workspace is defined in `package.json:1-68`. Node is constrained at `package.json:23-26`, Yarn workspaces are listed at `package.json:51-68`, and the root `start` script runs server, frontend, and worker at `package.json:44`.

Useful commands:

```powershell
yarn install
yarn start
npx nx typecheck twenty-front
npx nx typecheck twenty-server
npx nx test twenty-front
npx nx test twenty-server
```

## Reading Order

1. `packages/twenty-front/src/index.tsx:1-14`
2. `packages/twenty-front/src/modules/app/components/App.tsx:1-46`
3. `packages/twenty-front/src/modules/app/components/AppRouterProviders.tsx:39-96`
4. `packages/twenty-front/src/modules/app/hooks/useCreateAppRouter.tsx:109-260`
5. `packages/twenty-front/src/modules/users/hooks/useLoadCurrentUser.ts:24-153`
6. `packages/twenty-front/src/modules/apollo/services/apollo.factory.ts:98-365`
7. `packages/twenty-server/src/main.ts:1-96`
8. `packages/twenty-server/src/app.module.ts:49-146`
9. `packages/twenty-server/src/modules/dashboard/controllers/dashboard.controller.ts:1-25`
10. `packages/twenty-server/src/engine/api/rest/core/controllers/rest-api-core.controller.ts:24-163`

## First Tasks

- Explain frontend boot from `index.tsx:10-14` to `AppRouterProviders.tsx:72`.
- Trace current user loading through `useLoadCurrentUser.ts:48-65`.
- Trace a simple backend route through `dashboard.controller.ts:19-24`.
- Trace one metadata write through `view-sort.service.ts:63-100`.

## Vocabulary

- Workspace: tenant/customer context attached through middleware at `app.module.ts:114-146`.
- Metadata: configuration that defines CRM objects, views, fields, webhooks, layouts, and permissions.
- Atom: Jotai state unit created through `createAtomState.ts:27-79`.
- Apollo link: request pipeline step composed at `apollo.factory.ts:343-356`.
- Guard: Nest authorization/authentication gate, such as dashboard guards at `dashboard.controller.ts:11-13`.
- DTO: typed API response object, such as `DuplicatedDashboardDTO` imported at `dashboard.controller.ts:7`.

## Ask A Senior When Stuck

- "Is this state local, global, cached, URL-owned, or backend-owned?"
- "Is this endpoint GraphQL, metadata GraphQL, dynamic REST, or product REST?"
- "Which middleware/guard proves workspace access?"
- "Is this generated file safe to edit?"
- "Does this metadata change require migration validation?"

---

# Doc 2 - Architecture Guide For New Engineers

## Mental Model

Twenty is a metadata-driven CRM workspace. The frontend is a provider shell plus routed feature modules. The backend is a Nest engine that attaches workspace context, exposes GraphQL/REST/MCP APIs, and routes many metadata changes through validation and workspace migration services.

## Architecture Diagram

```text
Browser
  -> packages/twenty-front/src/index.tsx:10-14
  -> App providers at App.tsx:20-44
  -> RouterProvider at AppRouter.tsx:17-19
  -> AppRouterProviders at AppRouterProviders.tsx:43-96
  -> Route pages from useCreateAppRouter.tsx:109-260
  -> ApolloFactory at apollo.factory.ts:98-365
  -> Backend Nest app at main.ts:30-93
  -> AppModule at app.module.ts:49-146
  -> GraphQL / REST / MCP API modules
  -> Guards + middleware
  -> Services
  -> TypeORM / workspace data sources / metadata migrations
```

## State Decisions

- Global user/workspace state lives in Jotai: `currentUserState.ts:18-21`.
- Server responses flow through Apollo clients: `apollo.factory.ts:359-365`.
- Route state lives in React Router: `useCreateAppRouter.tsx:109-260`.
- Durable CRM data lives in backend services and data sources.

## API Overview

- GraphQL configured at `app.module.ts:52-56`.
- Metadata GraphQL module imported at `app.module.ts:68`.
- Admin GraphQL module imported at `app.module.ts:69`.
- REST module imported at `app.module.ts:70`.
- Dynamic REST object routes live at `rest-api-core.controller.ts:24-163`.
- Dashboard duplication route lives at `dashboard.controller.ts:19-24`.
- View sort metadata routes live at `view-sort.controller.ts:38-126`.

## Full-Stack Data Flow

Current user flow:

```text
AppRouterProviders mounts auth/runtime providers
  -> useLoadCurrentUser calls GetCurrentUserDocument at useLoadCurrentUser.ts:48-51
  -> Apollo adds auth headers at apollo.factory.ts:126-138
  -> backend middleware hydrates request at app.module.ts:114-146
  -> response returns currentUser
  -> useLoadCurrentUser writes atoms at useLoadCurrentUser.ts:65-128
  -> AppRouter reads currentUserState at AppRouter.tsx:10-14
```

---

# Doc 3 - Code Review Guide

## Ten Review Checkpoints

1. Boot/provider changes: inspect `App.tsx:20-44` and `AppRouterProviders.tsx:43-96`.
2. Route changes: inspect `useCreateAppRouter.tsx:109-260`.
3. Auth state changes: inspect `currentUserState.ts:4-21` and `useLoadCurrentUser.ts:65-128`.
4. Apollo changes: inspect `apollo.factory.ts:98-365`.
5. Token renewal changes: inspect `apollo.factory.ts:41-44` and `apollo.factory.ts:162-210`.
6. Middleware changes: inspect `main.ts:56-88` and `app.module.ts:114-146`.
7. Guard changes: inspect controller decorators such as `dashboard.controller.ts:11-13`.
8. Dynamic REST changes: inspect `rest-api-core.controller.ts:24-163`.
9. Metadata mutations: inspect `view-sort.service.ts:63-100`.
10. Generated type changes: inspect generated GraphQL files and source documents together.

## TypeScript Checks

- Is a generated type being narrowed intentionally, like `CurrentUser` at `currentUserState.ts:4-16`?
- Are JSON/Upload `any` values validated at runtime after `generated-metadata/graphql.ts:14-19`?
- Do controller body types match GraphQL/input DTO types?
- Do hook return types still match mutation return types, like `usePerformViewSortAPIPersist.ts:30-37`?

## Common Mistakes

- Editing generated files directly.
- Adding frontend fetches outside Apollo and losing auth headers.
- Adding controller methods without guards/filters.
- Writing metadata directly instead of using migration validation.
- Duplicating global state already owned by Jotai or Apollo.

## Sensitive Areas

- Security: `engine/guards`, `app.module.ts:114-146`, `rest-api-core.controller.ts:24-26`, `webhook.controller.ts:28-39`.
- Performance: `AppRouterProviders.tsx:43-96`, `useApolloFactory.ts:45-119`, metadata recompute services.
- Data integrity: workspace migration services and flat metadata utilities.

---

# Doc 4 - Debugging Guide

## React Rendering And State

- Blank screen: inspect `index.tsx:10-14`, `App.tsx:20-44`, `AppRouterProviders.tsx:43-96`.
- Wrong route: inspect `useCreateAppRouter.tsx:109-260`.
- Wrong admin visibility: inspect `AppRouter.tsx:10-18`.
- Wrong user/workspace: inspect `useLoadCurrentUser.ts:48-128`.

## Runtime Type Mismatches

- Start with generated type: `generated-metadata/graphql.ts:14-19`.
- Find consumer/cast, such as `useLoadCurrentUser.ts:90`.
- Add runtime validation where JSON crosses into UI logic.

## Node API And Async Issues

- Request setup: `main.ts:56-88`.
- Route middleware: `app.module.ts:114-146`.
- Simple route: `dashboard.controller.ts:19-24`.
- Dynamic REST route: `rest-api-core.controller.ts:24-163`.
- Metadata route: `view-sort.controller.ts:87-110`.

## Database Query Problems

- Identify whether data is core metadata, workspace metadata, or workspace record data.
- Inspect repository/service path, like `view-sort.service.ts:27-31`.
- Inspect migration/cache path, like `view-sort.service.ts:63-100`.

## Symptom Map

| Symptom | First files |
|---|---|
| Signed out unexpectedly | `apollo.factory.ts:162-210`, `useApolloFactory.ts:68-87` |
| Redirect loop | `WorkspaceProviderEffect.tsx:44-87` |
| View changes disappear | `usePerformViewSortAPIPersist.ts:21-223`, `view-sort.service.ts:63-100` |
| Permission failure | Controller guards and `app.module.ts:114-146` |
| Upload failure | `main.ts:66-88`, `apollo.factory.ts:98-101` |

---

# Doc 5 - Change Playbook

## Add A New Field To An Existing Entity

1. Database/entity/metadata definition under backend module.
2. DTO/input files near the backend module.
3. Service conversion/migration path, like `view-sort.service.ts:56-78`.
4. Controller/resolver response shape.
5. Frontend GraphQL document.
6. Generated frontend types.
7. State or UI consumer.
8. Tests.

## Add A New API Endpoint End To End

1. Choose API family: GraphQL, metadata GraphQL, dynamic REST, product REST.
2. Add controller/resolver with guards/filters, following `dashboard.controller.ts:11-24`.
3. Add service method.
4. Add DTO/input.
5. Register module if needed.
6. Add frontend query/mutation/hook.
7. Add tests for auth, success, error.

## Add A New React Page With Data Fetching

1. Add page under `packages/twenty-front/src/pages`.
2. Add route in `useCreateAppRouter.tsx:109-260`.
3. Add GraphQL document/hook.
4. Use Apollo from provider tree.
5. Add loading/error/empty states.
6. Add tests/story.

## Add A New Role Or Permission

1. Shared permission constant/type.
2. Backend guard or permission check.
3. Current user/workspace query update.
4. Generated frontend types.
5. State fan-out in `useLoadCurrentUser.ts:75-84`.
6. UI route/button visibility.
7. Backend authorization tests.

## Add A Backend Validation Rule

1. DTO/input validation.
2. Service-level validation if it needs workspace data.
3. Exception/filter mapping.
4. Frontend error display.
5. Tests for valid, invalid, unauthorized.

## Add A Filter Or Query Parameter

1. Controller `@Query`, like `view-sort.controller.ts:52`.
2. Service method.
3. Repository/query indexes.
4. Frontend query variables.
5. Generated types.
6. Tests for filter behavior and workspace isolation.

---

# Doc 6 - Senior Ownership Notes

## Critique

What is strong:

- Clear boot chain from `index.tsx:10-14` to `main.ts:30-93`.
- Central API transport in `apollo.factory.ts:98-365`.
- Strong backend modularity in `app.module.ts:49-146`.
- Metadata changes use migration validation, as shown in `view-sort.service.ts:63-78`.

Known debt:

- Provider tree complexity at `AppRouterProviders.tsx:43-96`.
- Dynamic REST readability cost at `rest-api-core.controller.ts:24-163`.
- JSON/Upload `any` in generated types at `generated-metadata/graphql.ts:14-19`.
- Manual REST body shapes can drift from DTOs, as seen in `webhook.controller.ts:65-83`.

## Risks

- Performance: provider breadth, Apollo recreation, metadata recompute.
- Security: guard removal, dynamic REST bypass, webhook permission mistakes.
- Testing: concurrent token renewal, permission matrix, migration failure behavior.

## Refactor Plan

| Priority | Change | Effort | Impact |
|---:|---|---|---|
| 1 | Add tests for Apollo token renewal dedupe | M | Reduces auth incident risk |
| 2 | Document/generate REST route inventory | M | Improves review safety |
| 3 | Split `AppRouterProviders` by concern | M | Improves onboarding and performance review |
| 4 | Add runtime validation for generated JSON consumers | M | Reduces runtime type bugs |
| 5 | Add permission matrix tests for dynamic REST and webhooks | L | Reduces security risk |

---

# Doc 7 - Interview Walkthrough

## 60-Second Non-Technical Summary

Twenty is an open-source CRM that helps teams manage workspace data like people, companies, dashboards, workflows, settings, and integrations. Its distinctive idea is that much of the workspace is metadata-driven, so the app can adapt objects, fields, views, layouts, and permissions instead of hard-coding everything.

## 60-Second Technical Summary

Twenty is an Nx monorepo with a React/Vite frontend and NestJS backend. The frontend boots through `index.tsx:10-14`, composes global providers in `App.tsx:20-44`, routes pages through `useCreateAppRouter.tsx:109-260`, and centralizes API transport in `apollo.factory.ts:98-365`. The backend boots through `main.ts:27-96`, composes GraphQL/REST/MCP/ORM/modules in `app.module.ts:49-146`, and handles many metadata changes through migration-backed services like `view-sort.service.ts:63-100`.

## Senior Architecture Explanation

The app is organized around workspace context. The frontend loads user/workspace state via `useLoadCurrentUser.ts:48-128`, Apollo sends auth and workspace-related headers through `apollo.factory.ts:126-138`, and the backend attaches request context through middleware in `app.module.ts:114-146`. Once context exists, routes and services can enforce permissions and load metadata-driven CRM behavior.

## Feature Walkthrough

View sort persistence starts in a frontend hook at `usePerformViewSortAPIPersist.ts:21-223`, sends generated GraphQL mutations through Apollo, reaches backend metadata handlers such as `view-sort.controller.ts:87-110`, and runs migration-backed persistence through `view-sort.service.ts:56-100`.

## Tradeoff To Discuss

Dynamic metadata-driven architecture makes custom CRM workspaces possible, but it makes local reasoning harder. `RestApiCoreController` at `rest-api-core.controller.ts:24-163` is flexible, but reviewers must understand runtime path parsing and permission checks rather than just reading one static route per resource.

## Bug Found And Debug Path

Potential bug class: concurrent token renewal. Start at `apollo.factory.ts:41-44` for shared `renewalPromise`, then inspect renewal handling at `apollo.factory.ts:162-210`. The correct regression test simulates two simultaneous unauthenticated responses and asserts only one renewal request happens.

## Improvement Next

Split `AppRouterProviders.tsx:43-96` into named provider groups while preserving behavior. This would make onboarding easier, reduce provider-order fear, and create clearer performance review boundaries.

