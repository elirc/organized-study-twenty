# 07 - Doc 3: Senior Ownership Guide

## Known Debt And Risk

1. Provider stack density: `AppRouterProviders` nests many concerns at `packages/twenty-front/src/modules/app/components/AppRouterProviders.tsx:44`. Ownership risk is boot regressions that look unrelated to the changed provider.
2. Dynamic REST complexity: wildcard routes in `RestApiCoreController` at `packages/twenty-server/src/engine/api/rest/core/controllers/rest-api-core.controller.ts:31` require parser/handler inspection for every endpoint behavior.
3. JSON scalar looseness: generated metadata types use `any` for JSON fields at `packages/twenty-front/src/generated-metadata/graphql.ts:16`.
4. Metadata mutation indirection: services like `ViewSortService` run flat-map and migration logic at `packages/twenty-server/src/engine/metadata-modules/view-sort/services/view-sort.service.ts:63`; powerful, but hard to onboard.
5. Permission-sensitive endpoints: `WebhookController` depends on settings permission guards at `packages/twenty-server/src/engine/metadata-modules/webhook/controllers/webhook.controller.ts:29`.

## Performance Risks

- Apollo client recreation or cache clearing if `useApolloFactory` dependencies are changed incorrectly at `packages/twenty-front/src/modules/apollo/hooks/useApolloFactory.ts:45`.
- Metadata map recomputation after writes at `packages/twenty-server/src/engine/metadata-modules/view-sort/services/view-sort.service.ts:87`.
- Raw database cron queries under workflow jobs, such as `packages/twenty-server/src/modules/workflow/workflow-trigger/automated-trigger/crons/jobs/workflow-cron-trigger-cron.job.ts:154`.

## Security Risks

- Removing `JwtAuthGuard`, `WorkspaceAuthGuard`, or `CustomPermissionGuard` from `RestApiCoreController` at `packages/twenty-server/src/engine/api/rest/core/controllers/rest-api-core.controller.ts:25`.
- Weakening webhook settings permissions at `packages/twenty-server/src/engine/metadata-modules/webhook/controllers/webhook.controller.ts:29`.
- Mishandling token renewal in `ApolloFactory` around `packages/twenty-front/src/modules/apollo/services/apollo.factory.ts:190`.
- Failing to enforce upload limits configured in `packages/twenty-server/src/main.ts:66`.

## Testing Gaps To Close First

1. Concurrent token renewal behavior in `packages/twenty-front/src/modules/apollo/services/apollo.factory.ts:41`.
2. Role/permission matrix around `RestApiCoreController` at `packages/twenty-server/src/engine/api/rest/core/controllers/rest-api-core.controller.ts:24`.
3. Metadata migration failure handling in `ViewSortService` at `packages/twenty-server/src/engine/metadata-modules/view-sort/services/view-sort.service.ts:80`.

## Refactor Priority List

| Priority | Change | Why first |
|---:|---|---|
| 1 | Document and test provider order in `AppRouterProviders`. | This reduces frontend boot fear before larger refactors. |
| 2 | Create a generated API surface inventory for REST controllers/resolvers. | This makes review safer for dynamic APIs. |
| 3 | Add runtime schemas for JSON fields at consumption boundaries. | This addresses generated `any` without fighting codegen. |
| 4 | Write a metadata mutation contributor guide. | View sort and webhook services repeat a pattern that juniors need to learn. |
| 5 | Add high-value security regression tests. | Permission regressions have high blast radius. |

