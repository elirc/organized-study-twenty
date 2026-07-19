# 02 Mid-Level Engineer

## Architecture diagram
```text
React root (index.tsx)
  -> App shell providers (App.tsx)
    -> Router providers + effects (AppRouterProviders.tsx)
      -> Route graph (useCreateAppRouter.tsx)
        -> Feature modules/pages
          -> Apollo transport + token handling (apollo.factory.ts)
            -> GraphQL /metadata + /graphql
            -> REST /rest/*

Nest bootstrap (main.ts)
  -> AppModule imports (GraphQL, REST, MCP, modules)
  -> Middleware per surface (app.module.ts configure)
  -> Guards + filters (controller/resolver level)
  -> Services (view-sort.service.ts etc.)
  -> TypeORM entities + repositories (view-sort.entity.ts)
  -> Postgres/Redis/ClickHouse config (twenty-config + datasource)
```

## Type system deep dive
- Frontend identity type narrowing through `CurrentUser` pick contract (`currentUserState.ts:4-20`) avoids overexposing raw GraphQL model in local state.
- API mutation variables typed by generated GraphQL documents (`usePerformViewSortAPIPersist.ts:11-19`, `32-36`, `78-83`).
- Cross-layer enum contract through shared package: `ViewSortDirection` used by entity and controller input (`view-sort.entity.ts:44-50`, `view-sort.controller.ts:103-108`).
- `any` risk appears mainly where generated objects are cast (`useLoadCurrentUser.ts:81-83`) and should be reviewed per PR.

## State management deep dive (current user)
Flow:
1. Query executes with Apollo client (`useLoadCurrentUser.ts:48-51`).
2. Primary identity set (`65`).
3. Workspace-related slices distributed (`67-85`, `103-121`).
4. Locale/theme preferences initialized (`96-100`).
5. Workspace domain memory persisted (`123-127`).

State flow diagram:
```text
GetCurrentUser query -> user object
  -> currentUserState
  -> currentWorkspaceMembersState
  -> currentUserWorkspaceState
  -> currentWorkspaceMemberState
  -> currentWorkspaceState
  -> workspaceAuthBypassProvidersState
```

## API contract map (representative high-value surfaces)
- REST dashboards duplicate: `POST /rest/dashboards/:id/duplicate` (`dashboard.controller.ts:11-24`), guards `JwtAuthGuard`, `WorkspaceAuthGuard`, `NoPermissionGuard`.
- REST metadata viewSorts:
  - `GET /rest/metadata/viewSorts` (`view-sort.controller.ts:48-59`)
  - `GET /rest/metadata/viewSorts/:id` (`61-85`)
  - `POST /rest/metadata/viewSorts` (`87-97`)
  - `PATCH /rest/metadata/viewSorts/:id` (`99-110`)
  - `DELETE /rest/metadata/viewSorts/:id` (`112-124`)
- GraphQL mirrors same domain in metadata API (`view-sort.resolver.ts:29-103`).

Rating:
- Consistency: 4/5
- Type safety: 4/5
- Error handling consistency: 3.5/5

## Component interaction map
- `App` -> `AppRouter` via composition (`App.tsx:36`).
- `AppRouter` -> `useCreateAppRouter` for route tree creation (`AppRouter.tsx:18`).
- `AppRouterProviders` -> `Outlet` to render current route (`AppRouterProviders.tsx:72`).
- `useLoadCurrentUser` -> Jotai states consumed by many feature components.

## End-to-end feature trace (View Sort)
1. UI mutation hook dispatch (`usePerformViewSortAPIPersist.ts:46-52`, `92-98`, `139-145`).
2. GraphQL metadata call with typed variables (`11-19`).
3. Resolver/controller authorize request (`view-sort.resolver.ts:52-99` or `view-sort.controller.ts:87-124`).
4. Service builds migration payload (`view-sort.service.ts:63-78`, `131-146`, `200-217`).
5. Migration validate/run service executes.
6. Flat maps recomputed and DTO returned (`87-100`, `155-169`, `226-240`).
7. Frontend mutation promise resolves and local UX handles errors (`57-71`, `104-118`, `151-165`).

## Diff-reading exercise
Hypothetical change: add `subFieldName` update support in PATCH payload validation and UI editing.
Read order:
1. `view-sort.entity.ts:52-53`
2. `update-view-sort.input.ts`
3. `view-sort.controller.ts:99-109`
4. `view-sort.service.ts:125-169`
5. Frontend mutation variable typing and forms.

Review questions:
- Are enum and optional field constraints preserved?
- Does partial update accidentally null existing value?
- Are permission guards unchanged?

## Mid-level Socratic checkpoint
1. Why is state split across multiple Jotai atoms instead of one giant user atom?
2. How does middleware order in `AppModule` affect auth context?
3. What bug is prevented by `ViewSortDirection` enum in entity and controller?
4. Why is create/update/delete routed through migration validator instead of direct repository save?
5. Which part of token renewal flow is most race-condition sensitive?
6. What part of this stack is most fragile under schema-version drift?
7. If you add a new metadata endpoint, where must guards/filters be aligned?
8. What is your PR review order for a view-sort change?

## How to self-grade
Strong answers connect frontend and backend behavior, include at least one concrete failure case, and cite at least one file/line per answer.
