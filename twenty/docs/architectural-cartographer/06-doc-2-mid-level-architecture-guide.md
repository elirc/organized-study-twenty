# 06 - Doc 2: Mid-Level Architecture Guide

## Architecture Summary

The frontend is a React/Vite app inside an Nx monorepo. It mounts at `packages/twenty-front/src/index.tsx:10`, composes global providers in `packages/twenty-front/src/modules/app/components/App.tsx:20`, composes runtime route providers in `packages/twenty-front/src/modules/app/components/AppRouterProviders.tsx:44`, and maps URLs to lazy pages in `packages/twenty-front/src/modules/app/hooks/useCreateAppRouter.tsx:109`.

The backend is a NestJS app. It starts in `packages/twenty-server/src/main.ts:27`, creates the app from `AppModule` at `packages/twenty-server/src/main.ts:30`, and composes GraphQL, REST, MCP, ORM, core engine, business modules, and middleware in `packages/twenty-server/src/app.module.ts:49`.

## State Management Model

Client state is primarily Jotai plus Apollo cache. Jotai store creation lives in `packages/twenty-front/src/modules/ui/utilities/state/jotai/jotaiStore.ts:4`, and the store is provided at `packages/twenty-front/src/modules/app/components/App.tsx:20`. Atom helpers are standardized by `createAtomState` in `packages/twenty-front/src/modules/ui/utilities/state/jotai/utils/createAtomState.ts:27`.

Important example: `currentUserState` is defined at `packages/twenty-front/src/modules/auth/states/currentUserState.ts:18`, loaded by `useLoadCurrentUser` at `packages/twenty-front/src/modules/users/hooks/useLoadCurrentUser.ts:48`, and consumed by `AppRouter` at `packages/twenty-front/src/modules/app/components/AppRouter.tsx:10`.

## API Contract Model

Frontend API transport uses Apollo. `ApolloProvider` creates a metadata Apollo client at `packages/twenty-front/src/modules/apollo/components/ApolloProvider.tsx:13`, and `useApolloFactory` defaults to `/graphql` while allowing URI overrides at `packages/twenty-front/src/modules/apollo/hooks/useApolloFactory.ts:45`. `ApolloFactory` builds auth headers at `packages/twenty-front/src/modules/apollo/services/apollo.factory.ts:111`, handles token renewal at `packages/twenty-front/src/modules/apollo/services/apollo.factory.ts:162`, and composes REST/upload/stream links at `packages/twenty-front/src/modules/apollo/services/apollo.factory.ts:343`.

Backend API families:

- `/graphql`: configured through `GraphQLModule.forRootAsync` at `packages/twenty-server/src/app.module.ts:52`.
- `/metadata`: handled by metadata GraphQL modules imported at `packages/twenty-server/src/app.module.ts:68`.
- `/admin-panel`: handled by admin panel GraphQL module at `packages/twenty-server/src/app.module.ts:69`.
- `/rest/*path`: dynamic object REST API in `packages/twenty-server/src/engine/api/rest/core/controllers/rest-api-core.controller.ts:24`.
- `/rest/metadata/viewSorts`: metadata REST API in `packages/twenty-server/src/engine/metadata-modules/view-sort/controllers/view-sort.controller.ts:38`.
- `/rest/webhooks` and `/rest/metadata/webhooks`: webhook REST API in `packages/twenty-server/src/engine/metadata-modules/webhook/controllers/webhook.controller.ts:28`.

## Full-Stack Feature Trace: Current User Boot

1. Browser renders `<App />` at `packages/twenty-front/src/index.tsx:14`.
2. `App` provides Jotai store at `packages/twenty-front/src/modules/app/components/App.tsx:20`.
3. `AppRouterProviders` mounts Apollo and auth providers at `packages/twenty-front/src/modules/app/components/AppRouterProviders.tsx:44`.
4. `useLoadCurrentUser` sends `GetCurrentUserDocument` at `packages/twenty-front/src/modules/users/hooks/useLoadCurrentUser.ts:48`.
5. Apollo adds auth and workspace headers in `packages/twenty-front/src/modules/apollo/services/apollo.factory.ts:111`.
6. Backend GraphQL request context is hydrated through middleware in `packages/twenty-server/src/app.module.ts:116`.
7. Frontend writes user/workspace/permission atoms in `packages/twenty-front/src/modules/users/hooks/useLoadCurrentUser.ts:65`.
8. `AppRouter` reads `currentUserState` and gates admin routes at `packages/twenty-front/src/modules/app/components/AppRouter.tsx:12`.

## Architecture Reading Habit

For every feature, ask:

1. Which route/page starts it?
2. Which provider makes its hooks legal?
3. Which atom or Apollo cache entry stores its state?
4. Which GraphQL document or REST directive sends the request?
5. Which Nest guard/middleware proves the user/workspace?
6. Which service owns the business rule?
7. Which repository, data source, or migration persists it?

