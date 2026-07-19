# Architectural Cartographer (Codex 5.3 Fresh Pass)

This suite is a top-down-to-bottom-up onboarding and ownership guide for the Twenty monorepo, written from scratch from direct repository inspection.

## How to use this suite
1. Read `00-reading-map.md` first.
2. Work through `01-junior-engineer.md`, `02-mid-level-engineer.md`, and `03-senior-engineer.md` in order.
3. Use `04-reference-suite.md` as your long-term field manual.
4. Use `JOURNAL.md` to understand why these anchors were selected.

## Checkpoint handling
Socratic checkpoints are included, but this guide does not pause. Each checkpoint includes a self-grading rubric so you can run active practice asynchronously.

## Anchor strategy
Primary anchors were chosen for architectural leverage:
- Frontend bootstrap and provider stack: `packages/twenty-front/src/index.tsx:1-14`, `packages/twenty-front/src/modules/app/components/App.tsx:1-46`, `packages/twenty-front/src/modules/app/components/AppRouterProviders.tsx:1-96`
- Frontend route graph: `packages/twenty-front/src/modules/app/hooks/useCreateAppRouter.tsx:109-267`
- Backend bootstrap and module wiring: `packages/twenty-server/src/main.ts:27-96`, `packages/twenty-server/src/app.module.ts:49-146`
- End-to-end metadata feature (view sort): `packages/twenty-front/src/modules/views/hooks/internal/usePerformViewSortAPIPersist.ts:21-223`, `packages/twenty-server/src/engine/metadata-modules/view-sort/controllers/view-sort.controller.ts:38-124`, `packages/twenty-server/src/engine/metadata-modules/view-sort/services/view-sort.service.ts:25-342`, `packages/twenty-server/src/engine/metadata-modules/view-sort/entities/view-sort.entity.ts:19-72`
