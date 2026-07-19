# Journal

## First-pass mental model
Twenty behaves like a metadata-driven CRM platform where both product behavior and user data shape are resolved through a layered startup pipeline: boot app, hydrate auth/workspace context, then gate feature surfaces by permissions and metadata readiness (`packages/twenty-front/src/modules/app/components/AppRouterProviders.tsx:44-95`, `packages/twenty-front/src/modules/metadata-store/effect-components/UserMetadataProviderInitialEffect.tsx:68-172`, `packages/twenty-server/src/app.module.ts:49-146`).

## Why these anchors
- `index.tsx` and `App.tsx` show frontend bootstrapping boundaries (`packages/twenty-front/src/index.tsx:10-14`, `packages/twenty-front/src/modules/app/components/App.tsx:20-45`).
- `useCreateAppRouter.tsx` reveals route and page-level composition (`packages/twenty-front/src/modules/app/hooks/useCreateAppRouter.tsx:109-267`).
- `main.ts` and `app.module.ts` expose backend middleware, module wiring, and protocol surfaces (`packages/twenty-server/src/main.ts:56-93`, `packages/twenty-server/src/app.module.ts:114-145`).
- `view-sort` path offers a realistic full-stack feature with auth guards, service orchestration, cache recompute, and persistence constraints.

## Where juniors get confused
- Provider depth in `AppRouterProviders` feels like accidental complexity before they realize each provider owns a dependency boundary.
- `ViewSortService` writes through workspace migration flows instead of direct repository writes for mutation operations (`packages/twenty-server/src/engine/metadata-modules/view-sort/services/view-sort.service.ts:63-85`, `131-153`, `200-224`).

## Where mid-levels should slow down
- Auth/token recovery in Apollo error link logic (`packages/twenty-front/src/modules/apollo/services/apollo.factory.ts:162-210`, `270-341`).
- Middleware ordering and route-specific guards (`packages/twenty-server/src/app.module.ts:114-145`).

## Where seniors should be skeptical
- Very large provider and middleware graphs can hide cross-cutting coupling.
- Mixed REST + GraphQL + metadata protocol surfaces can drift in error semantics.

## Setup confidence
Environment setup is partially explicit via `.env.example` and Nx targets (`packages/twenty-server/.env.example:1-100`, `packages/twenty-server/project.json:51-59`, `198-217`, `237-262`, `packages/twenty-front/.env.example:1-12`, `packages/twenty-front/project.json:37-43`).
