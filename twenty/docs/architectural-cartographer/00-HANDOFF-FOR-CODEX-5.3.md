# Handoff For Codex 5.3

You are continuing work in:

`c:\Users\Owner\Desktop\mastertwenty\twenty`

The user wants a large, labeled documentation response set under:

`docs/architectural-cartographer/`

This handoff exists so you can finish the documentation suite without starting from zero.

## User Goal

The user is using this open-source codebase to improve from junior toward mid-level and senior-level engineering skill. The docs should teach them how to reverse engineer a real React + TypeScript + Node/Nest codebase, review diffs, trace features end-to-end, debug intelligently, and explain the system under pressure.

They are comparing several generated responses, so this response set must be clearly labeled and self-contained.

## Critical Instruction From User

The original prompt asks for Socratic checkpoints that "wait for my answers." The user then clarified:

> At any breaks or pauses or checkpoints, you will not actually pause. You will just update the new documentation about what someone might have done in that situation and then what the best choice would have been, or why it was chosen.

So do not stop at checkpoints. Convert them into written learning checkpoint sections with questions, likely answers, mentor grading notes, and strong-answer examples.

The user also asked for a journal file inside the docs folder. Use:

`docs/architectural-cartographer/JOURNAL.md`

Put checkpoint reasoning, senior-thinking notes, and helpful upskilling observations there as you work.

## Repo State

- Full repo is cloned locally.
- Current branch: `main`
- Current commit: `6dd1e8a4712ab80ad76f2fbd4badf125f2f0b47d`
- Root docs folder did not exist before this handoff. It has now been created.
- No dev server is running.
- No tests have been run for this docs-only handoff.

## Architecture Facts Already Observed

This is an Nx monorepo.

Root workspace:

- `package.json` declares Yarn workspaces and scripts.
- `package.json` uses `yarn@4.13.0`.
- `package.json` requires Node `^24.5.0`.
- `package.json` `start` runs `twenty-server`, `twenty-front`, and then the server worker.
- `nx.json` configures Nx behavior.

Important packages:

- `packages/twenty-front` - main React frontend app.
- `packages/twenty-server` - main NestJS backend app.
- `packages/twenty-ui` - shared UI component library.
- `packages/twenty-shared` - shared domain utilities and types.
- `packages/twenty-client-sdk` - generated/client SDK related package.
- `packages/twenty-sdk` - SDK/build tooling for apps and logic/front components.
- `packages/twenty-docs` - docs tooling package, separate from the root `docs` folder being created for this learning artifact.
- `packages/twenty-website-new` - marketing/website app.
- `packages/twenty-apps` - app fixtures, examples, community and internal integrations.
- `packages/twenty-e2e-testing` - e2e test package.

## First Files To Read

Use these as the backbone of the reading map and entry point explanation:

1. `package.json`
   - Shows Yarn workspaces, Node/Yarn requirements, root scripts.
   - `package.json:44` shows `start`, which starts frontend, server, and worker together.
2. `nx.json`
   - Explains task orchestration and monorepo structure.
3. `packages/twenty-front/project.json`
   - `packages/twenty-front/project.json:37` defines the Vite dev server target.
   - `packages/twenty-front/project.json:66` defines typecheck.
   - `packages/twenty-front/project.json:67` defines lint.
   - `packages/twenty-front/project.json:109` defines test.
4. `packages/twenty-front/src/index.tsx`
   - `packages/twenty-front/src/index.tsx:1` imports `ReactDOM`.
   - `packages/twenty-front/src/index.tsx:3` imports `App`.
   - `packages/twenty-front/src/index.tsx:10` creates the React root.
   - `packages/twenty-front/src/index.tsx:14` renders `<App />`.
5. `packages/twenty-front/src/modules/app/components/App.tsx`
   - `packages/twenty-front/src/modules/app/components/App.tsx:16` activates i18n.
   - `packages/twenty-front/src/modules/app/components/App.tsx:20` provides the Jotai store.
   - `packages/twenty-front/src/modules/app/components/App.tsx:21` wraps the app in `AppErrorBoundary`.
   - `packages/twenty-front/src/modules/app/components/App.tsx:25` provides Lingui i18n.
   - `packages/twenty-front/src/modules/app/components/App.tsx:31` provides exception handling.
   - `packages/twenty-front/src/modules/app/components/App.tsx:36` hands off to `AppRouter`.
6. `packages/twenty-front/src/modules/app/components/AppRouter.tsx`
   - `packages/twenty-front/src/modules/app/components/AppRouter.tsx:10` reads `currentUserState`.
   - `packages/twenty-front/src/modules/app/components/AppRouter.tsx:12` derives admin page availability.
   - `packages/twenty-front/src/modules/app/components/AppRouter.tsx:17` renders `RouterProvider`.
7. `packages/twenty-front/src/modules/app/hooks/useCreateAppRouter.tsx`
   - Next frontend routing file to inspect. Not yet read in this handoff.
8. `packages/twenty-server/project.json`
   - `packages/twenty-server/project.json:51` defines server start.
   - `packages/twenty-server/project.json:118` defines worker start.
   - `packages/twenty-server/project.json:198` and later define database commands.
9. `packages/twenty-server/src/main.ts`
   - `packages/twenty-server/src/main.ts:27` defines `bootstrap`.
   - `packages/twenty-server/src/main.ts:30` creates the Nest Express application.
   - `packages/twenty-server/src/main.ts:56` installs session middleware.
   - `packages/twenty-server/src/main.ts:64` installs global exception handling.
   - `packages/twenty-server/src/main.ts:74` configures `/graphql` upload middleware.
   - `packages/twenty-server/src/main.ts:82` configures `/metadata` upload middleware.
   - `packages/twenty-server/src/main.ts:93` starts listening on configured `NODE_PORT`.
10. `packages/twenty-server/src/app.module.ts`
   - Next backend module composition file to inspect. Not yet read in this handoff.

## Recommended Output Shape

Keep the response set inside `docs/architectural-cartographer/`.

Suggested files:

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

The original prompt is huge. Do not try to produce everything in one giant unstructured file. Split it into the files above and keep cross-links clear.

## How To Continue Efficiently

Start with codebase discovery using fast commands:

```powershell
rg --files packages/twenty-front/src/modules/app
rg --files packages/twenty-front/src/modules/auth
rg --files packages/twenty-front/src/modules | Select-String "graphql|apollo|router|state|atom"
rg --files packages/twenty-server/src | Select-String "controller|resolver|module|service|guard|strategy"
rg "GraphQLModule|Apollo|TypeOrm|DataSource|@Resolver|@Controller|@UseGuards|@Mutation|@Query" packages/twenty-server/src
rg "useQuery|useMutation|gql|Apollo|atom|createState|createComponentState" packages/twenty-front/src
```

Then inspect:

- `packages/twenty-front/src/modules/app/hooks/useCreateAppRouter.tsx`
- `packages/twenty-front/src/modules/auth/states/currentUserState.ts`
- Apollo or GraphQL client setup in `packages/twenty-front/src`
- Main layout and object record modules in `packages/twenty-front/src/modules`
- `packages/twenty-server/src/app.module.ts`
- GraphQL API modules under `packages/twenty-server/src/engine/api/graphql`
- Auth modules under `packages/twenty-server/src/engine/core-modules/auth`
- Workspace/entity/metadata modules under `packages/twenty-server/src/engine`

## Documentation Quality Bar

Every codebase claim should cite a concrete file and ideally a line number. If a section is broad, cite the folder plus representative files.

Annotated code blocks should use comments as the primary explanation. For example:

```tsx
// This import is the handoff from browser DOM into React's renderer.
import ReactDOM from 'react-dom/client';

// App is the first application-owned component. Everything after this is app composition.
import { App } from '@/app/components/App';
```

For large files, do not paste hundreds of lines blindly. Reproduce meaningful slices when needed, but preserve the user's intent by annotating the entry points, three important components, and three important backend handlers in detail.

## Be Careful With Scope

The prompt asks for "every endpoint" and "every custom abstraction." This repo is large. A useful senior move is to:

- State the scan method and exact command.
- Generate inventories from search results.
- Group large surfaces by module.
- Mark generated or framework boilerplate separately.
- Cite representative files and explain how to continue the inventory.

Do not hand-wave. If something is too large for exhaustive prose, create a table or appendix and explain the method.

## Suggested Mental Model To Validate

Initial hypothesis:

Twenty is organized around a metadata-driven CRM workspace. The frontend boots into a provider-heavy React shell, uses Jotai and Apollo/GraphQL-style data access, and renders workspace features based on user/session/workspace metadata. The backend is a NestJS application whose entry point composes global middleware, session handling, GraphQL upload endpoints, config, logging, and exception handling before handing most domain behavior to modules under `engine`.

Validate or revise this after reading `useCreateAppRouter.tsx`, Apollo client setup, `app.module.ts`, GraphQL modules, and auth/workspace metadata modules.

## Journal Expectations

Update `JOURNAL.md` as you discover:

- Why a file belongs in the reading order.
- What a junior might misunderstand.
- What a mid-level engineer should notice.
- What a senior would inspect before approving a PR in that area.
- When a checkpoint would have paused, what the learner might answer, and what the stronger answer is.

## Do Not Forget

- The user asked for a handoff now, not the full guide from this model.
- The next model should finish the docs.
- Keep all Architectural Cartographer artifacts under `docs/architectural-cartographer/`.
- The final generated docs should be teaching documents, not just summaries.

