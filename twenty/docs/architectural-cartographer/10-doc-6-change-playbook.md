# 10 - Doc 6: Change Playbook

## Add A New Field To An Existing API Response

Example: add a field to current user.

Change order:

1. Backend resolver/schema source for `currentUser` under `packages/twenty-server/src`.
2. Frontend GraphQL fragment `packages/twenty-front/src/modules/users/graphql/fragments/userQueryFragment.ts`, because `GetCurrentUserDocument` uses it at `packages/twenty-front/src/modules/users/graphql/queries/getCurrentUser.ts:6`.
3. Regenerate frontend GraphQL output in `packages/twenty-front/src/generated-metadata/graphql.ts`.
4. Update `CurrentUser` if the field belongs in global state at `packages/twenty-front/src/modules/auth/states/currentUserState.ts:4`.
5. Update `useLoadCurrentUser` fan-out if the field affects workspace/auth state at `packages/twenty-front/src/modules/users/hooks/useLoadCurrentUser.ts:65`.
6. Add tests near the consuming hook/component.

Review question: is the field needed globally, or should it stay in Apollo cache?

## Add A New Entity To The Data Model

For metadata entities, follow the pattern of view sort or webhook.

Change order:

1. Entity under `packages/twenty-server/src/engine/metadata-modules/<entity>/entities`, like `ViewSortEntity` imported at `packages/twenty-server/src/engine/metadata-modules/view-sort/services/view-sort.service.ts:19`.
2. DTO/input files under `packages/twenty-server/src/engine/metadata-modules/<entity>/dtos`.
3. Flat entity conversion utilities under `packages/twenty-server/src/engine/metadata-modules/flat-<entity>/utils`.
4. Service that uses repositories, flat maps, and migration validation, like `ViewSortService` at `packages/twenty-server/src/engine/metadata-modules/view-sort/services/view-sort.service.ts:24`.
5. Controller/resolver, like `ViewSortController` at `packages/twenty-server/src/engine/metadata-modules/view-sort/controllers/view-sort.controller.ts:38`.
6. Module registration and `AppModule` import path through existing metadata module aggregation.
7. Frontend GraphQL documents and generated types.

Review question: is this core workspace record data, metadata, or instance-level data?

## Add A New Filter Or Query Parameter

Example: add a filter to view sorts.

Change order:

1. Controller query parameter, like `@Query('viewId') viewId?: string` at `packages/twenty-server/src/engine/metadata-modules/view-sort/controllers/view-sort.controller.ts:52`.
2. Service method that implements filtering, such as `findByViewId` called at `packages/twenty-server/src/engine/metadata-modules/view-sort/controllers/view-sort.controller.ts:55`.
3. Repository query and indexes if needed.
4. Frontend GraphQL/REST call site.
5. Tests for no filter, valid filter, invalid filter, and unauthorized filter.

Review question: does the filter preserve workspace isolation?

## Add A New User Role Or Permission Level

Change order:

1. Shared permission enum/constants under `packages/twenty-shared/src`.
2. Backend guards under `packages/twenty-server/src/engine/guards`, following examples like `SettingsPermissionGuard` used at `packages/twenty-server/src/engine/metadata-modules/webhook/controllers/webhook.controller.ts:19`.
3. Current user workspace query fragment and generated types, because permissions are read into state at `packages/twenty-front/src/modules/users/hooks/useLoadCurrentUser.ts:75`.
4. Frontend permission selectors, such as `packages/twenty-front/src/modules/auth/states/objectPermissionsFamilySelector.ts`.
5. Route/UI gates, such as `AppRouter` admin gate at `packages/twenty-front/src/modules/app/components/AppRouter.tsx:12`.
6. Permission matrix tests.

Review question: is enforcement backend-first, with frontend only hiding UI?

## Add A New React Page Or Route

Change order:

1. Add page under `packages/twenty-front/src/pages`.
2. Add or reuse `AppPath` from shared types used by `useCreateAppRouter` at `packages/twenty-front/src/modules/app/hooks/useCreateAppRouter.tsx:10`.
3. Add a lazy import beside other lazy imports at `packages/twenty-front/src/modules/app/hooks/useCreateAppRouter.tsx:19`.
4. Add a `<Route>` under the correct layout: `DefaultLayout` starts at `packages/twenty-front/src/modules/app/hooks/useCreateAppRouter.tsx:121`, `BlankLayout` starts at line 255.
5. Add navigation/menu entry if needed.
6. Add page test or Storybook story if this page owns meaningful behavior.

Review question: does this page require auth, metadata, workspace, or admin access?

## Add A New Backend Validation Rule

Change order:

1. Find the DTO/input type for the route, such as `CreateWebhookInput` imported at `packages/twenty-server/src/engine/metadata-modules/webhook/controllers/webhook.controller.ts:22`.
2. Add validation near DTO or conversion utility.
3. If validation depends on workspace metadata, add it in service or flat utility, not only DTO.
4. Ensure exception filters translate the error; example filters are attached at `packages/twenty-server/src/engine/metadata-modules/view-sort/controllers/view-sort.controller.ts:40`.
5. Add REST and/or GraphQL tests.

Review question: does validation happen before persistence and before migration execution?

## Add A New API Endpoint End To End

Change order:

1. Decide API family: GraphQL, metadata GraphQL, REST metadata, dynamic REST, or admin.
2. Add backend controller/resolver with guards. Use `WebhookResolver` at `packages/twenty-server/src/engine/metadata-modules/webhook/webhook.resolver.ts:24` or `WebhookController` at `packages/twenty-server/src/engine/metadata-modules/webhook/controllers/webhook.controller.ts:28` as patterns.
3. Add service method and repository/migration logic.
4. Add DTO/input/output type.
5. Register module if needed.
6. Add frontend GraphQL document or REST directive.
7. Regenerate types if GraphQL.
8. Wire hook/component.
9. Add tests: backend auth/permissions, service behavior, frontend happy/error path.

Review question: what authentication, workspace, permission, and error-filter guarantees does this endpoint need?

