# 08 - Doc 4: Code Review Guide

## Ten Review Checkpoints

1. Entry/provider impact: if a PR touches `App.tsx` or `AppRouterProviders.tsx`, verify provider order and boot effects around `packages/twenty-front/src/modules/app/components/AppRouterProviders.tsx:44`.
2. Auth state consistency: if a PR changes current user/workspace shape, inspect `packages/twenty-front/src/modules/users/hooks/useLoadCurrentUser.ts:65`.
3. Apollo transport behavior: if a PR touches API errors, inspect `packages/twenty-front/src/modules/apollo/services/apollo.factory.ts:270`.
4. Token renewal: verify `renewalPromise` behavior remains deduped at `packages/twenty-front/src/modules/apollo/services/apollo.factory.ts:41`.
5. Route visibility: if settings/admin pages change, inspect `packages/twenty-front/src/modules/app/components/AppRouter.tsx:12`.
6. Dynamic REST paths: if wildcard REST changes, inspect `RestApiCoreController` and `RestApiCoreService` at `packages/twenty-server/src/engine/api/rest/core/services/rest-api-core.service.ts:56`.
7. Guards: never skim controller guard changes such as `WebhookController` at `packages/twenty-server/src/engine/metadata-modules/webhook/controllers/webhook.controller.ts:29`.
8. Metadata migrations: if metadata changes, inspect service migration calls like `ViewSortService.createOne` at `packages/twenty-server/src/engine/metadata-modules/view-sort/services/view-sort.service.ts:63`.
9. Generated types: if GraphQL schema or documents change, verify generated files such as `packages/twenty-front/src/generated-metadata/graphql.ts` are updated.
10. Error filters: if API exceptions change, inspect filters attached at controller or global level, such as `packages/twenty-server/src/main.ts:64`.

## Common Mistakes

- Editing generated files directly instead of updating source schema/documents.
- Updating `currentUserState` fields without updating `USER_QUERY_FRAGMENT` or `useLoadCurrentUser`.
- Adding an API call outside Apollo and losing auth/app-version headers.
- Adding metadata CRUD that bypasses workspace migration validation.
- Adding a route without checking `AppPath`, lazy loading, and provider requirements.

## TypeScript Checks

- Does the PR introduce `any`, especially around generated JSON fields from `packages/twenty-front/src/generated-metadata/graphql.ts:16`?
- Does it narrow generated types intentionally, like `CurrentUser` does at `packages/twenty-front/src/modules/auth/states/currentUserState.ts:4`?
- Do DTOs and controller return types still match, like `Promise<ViewSortDTO>` at `packages/twenty-server/src/engine/metadata-modules/view-sort/controllers/view-sort.controller.ts:92`?
- Does a REST body duplicate a DTO shape, as `WebhookController.update` does at `packages/twenty-server/src/engine/metadata-modules/webhook/controllers/webhook.controller.ts:69`?

## Security-Sensitive Areas

- `packages/twenty-server/src/engine/guards`
- `packages/twenty-server/src/engine/core-modules/auth`
- `packages/twenty-server/src/engine/api/rest/core/controllers/rest-api-core.controller.ts`
- `packages/twenty-server/src/engine/metadata-modules/webhook`
- `packages/twenty-front/src/modules/apollo/services/apollo.factory.ts`

## Performance-Sensitive Areas

- `packages/twenty-front/src/modules/app/components/AppRouterProviders.tsx`
- `packages/twenty-front/src/modules/apollo/services/apollo.factory.ts`
- `packages/twenty-server/src/engine/metadata-modules/*/services`
- `packages/twenty-server/src/modules/workflow/**/crons`

## Data Integrity-Sensitive Areas

- Workspace migrations under `packages/twenty-server/src/engine/workspace-manager/workspace-migration`.
- Metadata flat entity utilities under `packages/twenty-server/src/engine/metadata-modules/flat-*`.
- Dynamic object REST handlers under `packages/twenty-server/src/engine/api/rest/core/handlers`.
- TypeORM entities under `packages/twenty-server/src/engine/metadata-modules/**/entities`.

