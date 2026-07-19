# 00 Reading Map

## Mental model in one paragraph
This system thinks in terms of workspace-scoped capability delivery. The frontend boots a provider pipeline that establishes configuration, auth, metadata, and runtime UI services before exposing route surfaces (`packages/twenty-front/src/modules/app/components/App.tsx:20-45`, `packages/twenty-front/src/modules/app/components/AppRouterProviders.tsx:44-95`). The backend mirrors this with a module graph and middleware chain that hydrates auth/workspace context per protocol surface (`packages/twenty-server/src/main.ts:56-93`, `packages/twenty-server/src/app.module.ts:114-145`). Most non-trivial writes are metadata-aware operations routed through migration/validation layers rather than direct table mutation (`packages/twenty-server/src/engine/metadata-modules/view-sort/services/view-sort.service.ts:63-85`, `131-153`).

## Top 10 file reading order
1. `packages/twenty-front/src/index.tsx:1-14`
Reason now: smallest frontend bootstrap surface.
Before: know React root rendering.
After: explain where UI control starts.

2. `packages/twenty-front/src/modules/app/components/App.tsx:1-46`
Reason now: first provider boundary.
Before: know root render call.
After: explain cross-cutting provider ordering.

3. `packages/twenty-front/src/modules/app/components/AppRouter.tsx:1-19`
Reason now: handoff from providers to routing.
Before: know app shell.
After: explain route creation dependency on user flags.

4. `packages/twenty-front/src/modules/app/hooks/useCreateAppRouter.tsx:109-267`
Reason now: full route map.
Before: know AppPath usage.
After: explain layout vs feature route surfaces.

5. `packages/twenty-server/src/main.ts:27-96`
Reason now: backend bootstrap and middleware registration.
Before: know Nest startup basics.
After: explain how HTTP server and upload/session middleware initialize.

6. `packages/twenty-server/src/app.module.ts:49-146`
Reason now: runtime module composition and per-route middleware.
Before: know bootstrap entry.
After: explain protocol-specific middleware stack.

7. `packages/twenty-front/src/modules/apollo/services/apollo.factory.ts:98-210`
Reason now: frontend API transport and auth renewal behavior.
Before: know auth/token context exists.
After: explain retry, error handling, and token refresh flow.

8. `packages/twenty-front/src/modules/users/hooks/useLoadCurrentUser.ts:47-135`
Reason now: user/workspace state hydration.
Before: know Apollo client exists.
After: explain where identity state enters frontend.

9. `packages/twenty-server/src/engine/metadata-modules/view-sort/controllers/view-sort.controller.ts:38-124`
Reason now: concrete guarded REST API.
Before: know backend guard pattern.
After: explain request path to service.

10. `packages/twenty-server/src/engine/metadata-modules/view-sort/services/view-sort.service.ts:34-342`
Reason now: business logic + metadata migration write pipeline.
Before: know controller inputs.
After: explain why writes are validated via migration layer.

## Three core data flows
1. Frontend boot and provider hydration flow.
Anchors: `packages/twenty-front/src/index.tsx:10-14`, `packages/twenty-front/src/modules/app/components/App.tsx:20-45`, `packages/twenty-front/src/modules/app/components/AppRouterProviders.tsx:44-95`.

2. Authenticated request and middleware context flow.
Anchors: `packages/twenty-server/src/main.ts:56-80`, `packages/twenty-server/src/app.module.ts:114-145`.

3. View sort mutation flow (UI mutation -> backend validation -> persistence metadata maps -> UI update).
Anchors: `packages/twenty-front/src/modules/views/hooks/internal/usePerformViewSortAPIPersist.ts:30-223`, `packages/twenty-server/src/engine/metadata-modules/view-sort/controllers/view-sort.controller.ts:87-124`, `packages/twenty-server/src/engine/metadata-modules/view-sort/services/view-sort.service.ts:34-241`.

## Pre-reading checklist (10)
1. Can you explain why `App.tsx` wraps `AppRouter` with multiple providers (`packages/twenty-front/src/modules/app/components/App.tsx:20-45`)?
2. Can you identify where lazy route loading starts (`packages/twenty-front/src/modules/app/hooks/useCreateAppRouter.tsx:19-107`)?
3. Can you describe how server sessions are initialized (`packages/twenty-server/src/main.ts:56`)?
4. Can you show where workspace context middleware is attached (`packages/twenty-server/src/app.module.ts:140-143`)?
5. Can you explain how current user state enters Jotai (`packages/twenty-front/src/modules/users/hooks/useLoadCurrentUser.ts:65-85`)?
6. Can you explain why token renewal is deduplicated (`packages/twenty-front/src/modules/apollo/services/apollo.factory.ts:41-45`, `190-210`)?
7. Can you point to REST guard usage in a feature controller (`packages/twenty-server/src/engine/metadata-modules/view-sort/controllers/view-sort.controller.ts:39-44`)?
8. Can you explain soft delete semantics for view sort (`packages/twenty-server/src/engine/metadata-modules/view-sort/entities/view-sort.entity.ts:64-65`)?
9. Can you explain route-level capability gating in settings (`packages/twenty-front/src/modules/app/hooks/useCreateAppRouter.tsx:238-244`)?
10. Can you describe the difference between delete vs destroy in service semantics (`packages/twenty-server/src/engine/metadata-modules/view-sort/services/view-sort.service.ts:172-301`)?

## Red flags checklist
1. Provider order changes without dependency reasoning (`packages/twenty-front/src/modules/app/components/AppRouterProviders.tsx:44-95`).
2. Silent auth failure paths that clear state without UX recovery (`packages/twenty-front/src/modules/apollo/services/apollo.factory.ts:184-199`).
3. Missing guard symmetry between REST and GraphQL endpoints (`packages/twenty-server/src/engine/metadata-modules/view-sort/controllers/view-sort.controller.ts:88-114`, `resolvers/view-sort.resolver.ts:52-92`).
4. Direct repository mutation introduced into metadata-managed writes (compare `createOne/updateOne/deleteOne` patterns).
5. Inconsistent soft-delete filtering (`deletedAt: IsNull()` expected in read methods: `view-sort.service.ts:303-338`).
6. Dynamic route additions without AppPath alignment (`packages/twenty-front/src/modules/app/hooks/useCreateAppRouter.tsx:121-253`).
7. Middleware order changes that break auth context hydration (`packages/twenty-server/src/app.module.ts:116-143`).
8. New `any`-typed state surface around user/workspace objects (`packages/twenty-front/src/modules/auth/states/currentUserState.ts:4-20`).
9. Upload/multipart paths introduced without size limits (`packages/twenty-server/src/main.ts:74-88`).
10. Missing tests around permission guards or migration-driven writes.
