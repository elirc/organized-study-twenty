# Architectural Cartographer Journal

This journal is the place to capture checkpoints, reasoning, tradeoffs, mentor notes, and senior-engineer thinking while producing the Architectural Cartographer documentation set.

## Working Principle

The original prompt asks for Socratic checkpoints that would normally pause and wait for the learner's answers. The user clarified that the documentation should not actually pause. Instead, every checkpoint should include:

- The questions the learner should answer.
- What a learner might reasonably answer at that stage.
- How a mentor would grade or respond.
- What a strong answer should include.
- Why the best next reading or debugging choice was chosen.

## Session Notes

### 2026-05-13

- Repository was cloned into `c:\Users\Owner\Desktop\mastertwenty\twenty`.
- The clone required Windows-specific handling:
  - `git config --global core.longpaths true` was set.
  - The repo directory was recreated with Windows case-sensitive directory support enabled.
  - This was needed because the repo contains very long paths and case-colliding paths such as `Price` and `price` under website assets.
- Verified checkout:
  - Branch: `main`
  - Commit: `6dd1e8a4712ab80ad76f2fbd4badf125f2f0b47d`
  - File count: `21,783`

### Initial Codebase Observations

- The repo is an Nx monorepo. Root workspace configuration lives in `package.json` and `nx.json`.
- The main frontend app package is `packages/twenty-front`.
- The main backend app package is `packages/twenty-server`.
- Important supporting packages include `packages/twenty-ui`, `packages/twenty-shared`, `packages/twenty-client-sdk`, `packages/twenty-sdk`, `packages/twenty-docs`, `packages/twenty-website-new`, and integration/app packages under `packages/twenty-apps`.
- Frontend entry point:
  - `packages/twenty-front/src/index.tsx:1` imports `ReactDOM`.
  - `packages/twenty-front/src/index.tsx:3` imports `App`.
  - `packages/twenty-front/src/index.tsx:10` creates the React root.
  - `packages/twenty-front/src/index.tsx:14` renders `<App />`.
- Frontend app provider stack:
  - `packages/twenty-front/src/modules/app/components/App.tsx:16` activates i18n before rendering.
  - `packages/twenty-front/src/modules/app/components/App.tsx:20` provides the Jotai store.
  - `packages/twenty-front/src/modules/app/components/App.tsx:21` wraps the app in an error boundary.
  - `packages/twenty-front/src/modules/app/components/App.tsx:25` provides Lingui i18n.
  - `packages/twenty-front/src/modules/app/components/App.tsx:31` provides exception handling.
  - `packages/twenty-front/src/modules/app/components/App.tsx:36` hands off to `AppRouter`.
- Frontend router handoff:
  - `packages/twenty-front/src/modules/app/components/AppRouter.tsx:1` imports `useCreateAppRouter`.
  - `packages/twenty-front/src/modules/app/components/AppRouter.tsx:2` imports `currentUserState`.
  - `packages/twenty-front/src/modules/app/components/AppRouter.tsx:10` reads the current user from Jotai state.
  - `packages/twenty-front/src/modules/app/components/AppRouter.tsx:12` derives whether admin pages are enabled.
  - `packages/twenty-front/src/modules/app/components/AppRouter.tsx:17` renders React Router's `RouterProvider`.
- Backend entry point:
  - `packages/twenty-server/src/main.ts:20` imports `AppModule`.
  - `packages/twenty-server/src/main.ts:27` defines `bootstrap`.
  - `packages/twenty-server/src/main.ts:28` sets PostgreSQL date parsing.
  - `packages/twenty-server/src/main.ts:30` creates the Nest Express application.
  - `packages/twenty-server/src/main.ts:46` retrieves `LoggerService`.
  - `packages/twenty-server/src/main.ts:47` retrieves `TwentyConfigService`.
  - `packages/twenty-server/src/main.ts:56` installs session middleware.
  - `packages/twenty-server/src/main.ts:64` installs the global unhandled exception filter.
  - `packages/twenty-server/src/main.ts:66` configures JSON body parsing.
  - `packages/twenty-server/src/main.ts:74` configures GraphQL upload middleware at `/graphql`.
  - `packages/twenty-server/src/main.ts:82` configures GraphQL upload middleware at `/metadata`.
  - `packages/twenty-server/src/main.ts:91` generates frontend config.
  - `packages/twenty-server/src/main.ts:93` listens on `NODE_PORT`.
  - `packages/twenty-server/src/main.ts:96` invokes `bootstrap`.

## Checkpoint Handling Pattern

When a section asks to "Wait for my answers", write it as a journaled learning checkpoint:

1. List the question.
2. Add a short "Why this question matters" note.
3. Add "A developing answer might say..."
4. Add "A strong answer should include..."
5. Add "Mentor correction / next step..."

This keeps the document interactive in spirit while satisfying the user's instruction not to pause.

## Mentor Voice Guidance

- Treat the learner as capable, not fragile.
- Explain how to think, not only what the code does.
- Show file-navigation choices explicitly: "I would open X before Y because..."
- Where the original prompt asks for line-by-line annotations, use annotated code blocks with comments as the main explanation.
- Prefer teaching through traces: entry point, router, state atom, API boundary, service, repository, test.
- Make review instincts explicit: "What could break?", "What assumption is this code making?", "Where would I look if production failed?"

## Open Work

The Architectural Cartographer guide has been expanded into numbered documents:

- `01-codebase-reading-map.md`
- `02-level-1-junior-engineer.md`
- `03-level-2-mid-level-engineer.md`
- `04-level-3-senior-engineer.md`
- `05-doc-1-junior-onboarding-guide.md`
- `06-doc-2-mid-level-architecture-guide.md`
- `07-doc-3-senior-ownership-guide.md`
- `08-doc-4-code-review-guide.md`
- `09-doc-5-debugging-guide.md`
- `10-doc-6-change-playbook.md`
- `11-doc-7-interview-walkthrough.md`

The docs intentionally convert "wait for my answers" prompts into active learning tables with likely answers and strong-answer expectations. The codebase is large enough that the API and abstraction inventories are framed as representative, high-signal maps rather than pretending every generated or dynamic route can be meaningfully pasted into one static guide.

### Senior Reasoning Notes Added During Buildout

- The best first mental model is "metadata-driven workspace CRM" because `AppRouterProviders`, `ApolloFactory`, `AppModule`, dynamic REST, and metadata services all point to workspace context as the central organizing idea.
- `AppRouterProviders` is the frontend's main onboarding hazard. It is correct to read it early because route bugs, auth bugs, metadata bugs, and UI-shell bugs can all surface through this provider stack.
- `ApolloFactory` is the most important frontend backend-boundary file because it owns auth headers, retry, token renewal, REST links, streaming links, upload links, app version handling, and error reporting.
- `RestApiCoreController` should be taught as a dynamic object API, not as a normal one-controller-per-resource REST design.
- Metadata services such as `ViewSortService` and `WebhookService` should be taught as migration-backed metadata mutation paths, not simple CRUD service classes.
