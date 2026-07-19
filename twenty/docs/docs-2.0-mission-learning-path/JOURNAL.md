# Docs 2.0 Journal

## Fresh Pass Notes

This documentation pass treats the repo as a fresh open-source codebase and uses the mission prompt as the organizing style.

## Key Discoveries

- The frontend boot chain is short at the DOM entry and much richer in the app shell: `packages/twenty-front/src/index.tsx:10-14` renders `App`, while `packages/twenty-front/src/modules/app/components/App.tsx:20-44` installs Jotai, error handling, i18n, icons, Helmet, click-outside context, and routing.
- The runtime route shell is `packages/twenty-front/src/modules/app/components/AppRouterProviders.tsx:39-96`. This is the best file for learning provider composition because it stacks Apollo, metadata loading, auth, workspace, SSE, command menu, UI overlays, and route effects above `<Outlet />`.
- The backend boot chain starts at `packages/twenty-server/src/main.ts:27-96`. `AppModule` at `packages/twenty-server/src/app.module.ts:49-146` is the architecture map.
- A useful simple component is `packages/twenty-front/src/modules/auth/components/Title.tsx:1-35`.
- A useful simple route is `packages/twenty-server/src/modules/dashboard/controllers/dashboard.controller.ts:1-25`.
- A useful complex hook is `packages/twenty-front/src/modules/views/hooks/internal/usePerformViewSortAPIPersist.ts:1-223`.
- A useful side-effect teaching file is `packages/twenty-front/src/modules/workspace/components/WorkspaceProviderEffect.tsx:44-87`.

## Checkpoint Convention

The prompt says to wait for answers. Because this deliverable is a folder of docs rather than an interactive lesson, every mission contains:

- Questions to answer before moving on.
- A self-grade rubric.
- Strong-answer markers.

The learner should stop at those sections and answer privately before reading the rubric.

