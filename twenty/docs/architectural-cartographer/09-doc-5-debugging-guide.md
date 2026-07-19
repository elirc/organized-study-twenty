# 09 - Doc 5: Debugging Guide

## React Rendering Issues

Start at the route. `useCreateAppRouter` maps `AppPath` values to lazy pages at `packages/twenty-front/src/modules/app/hooks/useCreateAppRouter.tsx:109`. If the URL is right but the page is blank, inspect `LazyRoute`, then the provider shell at `packages/twenty-front/src/modules/app/components/AppRouterProviders.tsx:44`, especially `MinimalMetadataGater` at line 53.

Symptom-first checklist:

- Blank app after boot: inspect `App.tsx:20`, `AppRouterProviders.tsx:44`, and browser console.
- Route not found: inspect `useCreateAppRouter.tsx:247`.
- Settings/admin page missing: inspect `AppRouter.tsx:12`.
- UI not reacting to user change: inspect `currentUserState.ts:18` and `useLoadCurrentUser.ts:65`.

## Runtime TypeScript-Type Mismatches

Generated types are compile-time contracts, not runtime validators. If a JSON field behaves unexpectedly, check `Scalars['JSON']` and `Scalars['JSONObject']` being `any` in `packages/twenty-front/src/generated-metadata/graphql.ts:16`.

Debug path:

1. Find the generated type in `packages/twenty-front/src/generated-metadata/graphql.ts`.
2. Find the GraphQL fragment/query that selects it.
3. Find where the app narrows or casts it, such as `ColorScheme` cast in `packages/twenty-front/src/modules/users/hooks/useLoadCurrentUser.ts:90`.
4. Add runtime validation at the consumption point.

## Node API Issues

For GraphQL issues, start at `packages/twenty-server/src/app.module.ts:52`, then find resolver/controller by operation name. For REST object issues, start at `RestApiCoreController` at `packages/twenty-server/src/engine/api/rest/core/controllers/rest-api-core.controller.ts:24`, then follow to `RestApiCoreService` at `packages/twenty-server/src/engine/api/rest/core/services/rest-api-core.service.ts:24`.

If a request is unauthorized:

1. Frontend headers: `packages/twenty-front/src/modules/apollo/services/apollo.factory.ts:111`.
2. Token renewal: `packages/twenty-front/src/modules/apollo/services/apollo.factory.ts:190`.
3. Backend middleware: `packages/twenty-server/src/app.module.ts:114`.
4. Controller guards: example `packages/twenty-server/src/engine/metadata-modules/webhook/controllers/webhook.controller.ts:29`.

## Database Query Problems

Core DB setup is in `packages/twenty-server/src/database/typeorm/typeorm.module.ts:9`. Workspace-aware data source behavior is in `packages/twenty-server/src/engine/twenty-orm/global-workspace-datasource/global-workspace-datasource.service.ts:16`. Metadata services often use repositories plus flat map caches, as `WebhookService` does at `packages/twenty-server/src/engine/metadata-modules/webhook/webhook.service.ts:27`.

Debug path:

1. Is this core metadata, workspace metadata, or workspace record data?
2. Is it using TypeORM repository, global workspace data source, or migration runner?
3. Does the query include `workspaceId` and `deletedAt` filters, like `WebhookService.findAll` at `packages/twenty-server/src/engine/metadata-modules/webhook/webhook.service.ts:48`?
4. Does cache recompute happen after mutation, like `ViewSortService.createOne` at `packages/twenty-server/src/engine/metadata-modules/view-sort/services/view-sort.service.ts:87`?

## Logs And Traces

Backend logging starts with `LoggerService` retrieved in `packages/twenty-server/src/main.ts:46`. REST object requests log method/path/workspace in `RestApiCoreController`, for example `packages/twenty-server/src/engine/api/rest/core/controllers/rest-api-core.controller.ts:62`.

Frontend GraphQL errors can go to Sentry from `sendToSentry` in `packages/twenty-front/src/modules/apollo/services/apollo.factory.ts:212`. In debug mode, the Apollo logger link can be included at `packages/twenty-front/src/modules/apollo/services/apollo.factory.ts:349`.

## Files To Check By Symptom

| Symptom | First files |
|---|---|
| Signed out unexpectedly | `apollo.factory.ts`, `useApolloFactory.ts`, `tokenPairState.ts`, backend auth guards. |
| Page missing | `useCreateAppRouter.tsx`, `AppPath` in `twenty-shared/types`, lazy page file. |
| Metadata changed but not persisted | Metadata controller, metadata service, flat utility, migration runner. |
| REST object API wrong result | `RestApiCoreController`, `RestApiCoreService`, path parsers, handler. |
| File upload fails | `main.ts` upload middleware, Apollo upload link, relevant upload hook. |
| Permission issue | Backend guard file, controller decorators, current user workspace permissions. |

