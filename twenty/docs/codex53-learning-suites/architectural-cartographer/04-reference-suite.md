# 04 Reference Suite

## Doc 1: Junior Onboarding Guide
- Start files: `index.tsx:1-14`, `App.tsx:1-46`, `main.ts:27-96`, `app.module.ts:49-146`.
- First tasks: run app locally, trace one route, trace one state hydration flow.
- Ask senior when stuck: provider ordering, workspace context, metadata write path.

## Doc 2: Mid-Level Architecture Guide
- System map: frontend provider/routing pipeline + backend module/middleware pipeline.
- Key flows: boot, auth context propagation, view-sort mutation loop.
- Contracts: shared enums + generated GraphQL types + guarded REST routes.

## Doc 3: Senior Ownership Guide
- Debt register: provider depth, migration-write cost, telemetry fragmentation.
- Risk matrix: auth renewal, guard parity, metadata invalidation correctness.
- Upgrade path: modularize providers, tighten API consistency checks, add end-to-end confidence tests.

## Doc 4: Code Review Guide
1. Confirm file/feature boundaries are preserved.
2. Confirm guard/filter symmetry on backend endpoints.
3. Confirm `deletedAt` soft-delete semantics in reads.
4. Confirm shared enums/types used over literals.
5. Confirm token/auth flow unaffected by networking changes.
6. Confirm migration-based metadata writes still used where expected.
7. Confirm locale and workspace context headers preserved in API calls.
8. Confirm route additions use `AppPath` and do not bypass wrappers.
9. Confirm tests cover permission and failure paths.
10. Confirm no accidental broad rerender surfaces.

## Doc 5: Debugging Guide
- Login loops: inspect `WorkspaceProviderEffect.tsx:44-87`, domain hooks.
- Missing user state: inspect `useLoadCurrentUser.ts:47-135` and GraphQL query success path.
- API auth failures: inspect `apollo.factory.ts:270-341`, backend guards.
- Metadata write failure: inspect `view-sort.controller.ts`, `view-sort.service.ts` validation branches.

## Doc 6: Change Playbook
- Add field to metadata entity: entity -> DTO/input -> service mapping -> controller/resolver -> frontend query/mutation typing -> UI.
- Add endpoint: controller/resolver + guard/filter + service + tests + frontend consumption.
- Add route/page: `AppPath` enum -> router map -> lazy page -> provider dependencies.

## Doc 7: Interview Walkthrough
- Non-technical 60s: “We run a customizable open-source CRM where each workspace can shape data and behavior safely.”
- Technical 60s: “React + Jotai + Apollo frontend, NestJS modular backend, metadata-driven write pipeline with workspace guards and migration validation.”
- Tradeoff: metadata safety vs write-path complexity.
- Bug story: token renewal race and dedupe strategy in Apollo layer.
- Next improvement: targeted metadata map invalidation and provider decomposition.
