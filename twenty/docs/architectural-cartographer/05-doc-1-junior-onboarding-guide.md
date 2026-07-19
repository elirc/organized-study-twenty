# 05 - Doc 1: Junior Onboarding Guide

## Setup Orientation

Start at the root `package.json`. It declares Yarn workspaces at `package.json:51`, Node `^24.5.0` at `package.json:23`, and the main app startup command at `package.json:44`. The main app pair is `packages/twenty-front` and `packages/twenty-server`, with supporting packages such as `packages/twenty-ui`, `packages/twenty-shared`, `packages/twenty-client-sdk`, and `packages/twenty-sdk`.

Use these commands as your first map:

```powershell
yarn install
yarn start
npx nx typecheck twenty-front
npx nx typecheck twenty-server
npx nx test twenty-front
npx nx test twenty-server
```

The frontend target is defined in `packages/twenty-front/project.json:37`. The backend target is defined in `packages/twenty-server/project.json:51`. The worker target is defined in `packages/twenty-server/project.json:118`.

## First Files To Read

Read in this order:

1. `package.json` for workspace/start context.
2. `packages/twenty-front/src/index.tsx` for browser entry.
3. `packages/twenty-front/src/modules/app/components/App.tsx` for global frontend shell.
4. `packages/twenty-front/src/modules/app/components/AppRouterProviders.tsx` for runtime providers.
5. `packages/twenty-front/src/modules/app/hooks/useCreateAppRouter.tsx` for route mapping.
6. `packages/twenty-front/src/modules/apollo/services/apollo.factory.ts` for frontend API transport.
7. `packages/twenty-front/src/modules/users/hooks/useLoadCurrentUser.ts` for auth/workspace state loading.
8. `packages/twenty-server/src/main.ts` for backend process startup.
9. `packages/twenty-server/src/app.module.ts` for backend module composition.
10. `packages/twenty-server/src/engine/api/rest/core/controllers/rest-api-core.controller.ts` for dynamic REST object APIs.

## First Tasks To Attempt

1. Trace current user loading from `GetCurrentUserDocument` in `packages/twenty-front/src/modules/users/hooks/useLoadCurrentUser.ts:48` to `AppRouter` route decisions in `packages/twenty-front/src/modules/app/components/AppRouter.tsx:12`.
2. Trace one REST metadata endpoint from `ViewSortController.create` at `packages/twenty-server/src/engine/metadata-modules/view-sort/controllers/view-sort.controller.ts:87` to `ViewSortService.createOne` at `packages/twenty-server/src/engine/metadata-modules/view-sort/services/view-sort.service.ts:34`.
3. Trace one Apollo error from `ErrorLink` at `packages/twenty-front/src/modules/apollo/services/apollo.factory.ts:270` to `onUnauthenticatedError` clearing state at `packages/twenty-front/src/modules/apollo/hooks/useApolloFactory.ts:68`.
4. Add or update a small test near an existing utility test, such as object-record tests under `packages/twenty-front/src/modules/object-record/utils/__tests__`.

## Vocabulary

- Workspace: the tenant/customer context for CRM data; backend middleware attaches it through `WorkspaceAuthContextMiddleware` in `packages/twenty-server/src/app.module.ts:118`.
- Metadata: definitions that shape objects, fields, views, sorts, roles, layouts, and webhooks; metadata modules live under `packages/twenty-server/src/engine/metadata-modules`.
- Object record: a CRM record such as a person/company/task represented through dynamic metadata; dynamic REST routes live in `packages/twenty-server/src/engine/api/rest/core/controllers/rest-api-core.controller.ts:24`.
- Atom: a small Jotai state container created through `createAtomState` at `packages/twenty-front/src/modules/ui/utilities/state/jotai/utils/createAtomState.ts:27`.
- Apollo link: one step in Apollo's request pipeline; auth, retry, REST, stream, and upload links are composed in `packages/twenty-front/src/modules/apollo/services/apollo.factory.ts:343`.
- Guard: a Nest access-control check, such as `SettingsPermissionGuard` in `packages/twenty-server/src/engine/metadata-modules/webhook/controllers/webhook.controller.ts:29`.

## What To Ask A Senior

Ask these when stuck:

1. "Is this data a workspace record, metadata, or frontend-only state?"
2. "Which provider makes this hook safe to call?"
3. "Is this API going through `/graphql`, `/metadata`, or dynamic `/rest`?"
4. "What guard or permission proves this action is allowed?"
5. "Does this metadata change need a workspace migration?"
6. "Is this generated file an input to edit or an output to regenerate?"

