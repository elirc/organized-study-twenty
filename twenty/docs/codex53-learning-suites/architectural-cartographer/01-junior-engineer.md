# 01 Junior Engineer

## Local setup and run
1. Install deps with Yarn 4 workspace setup (`package.json:26`, `42-47`).
2. Copy env files:
- Server: `packages/twenty-server/.env.example:1-100`
- Frontend: `packages/twenty-front/.env.example:1-12`
3. Start both app + worker from root: `npm run start` equivalent script is `yarn start` (`package.json:46`).
4. Backend-only dev: `nx run twenty-server:start` (`packages/twenty-server/project.json:51-59`).
5. Frontend-only dev: `nx run twenty-front:start` (`packages/twenty-front/project.json:37-43`).
6. DB init/reset available via Nx targets (`packages/twenty-server/project.json:206-217`, `237-262`).

## Folder orientation (top-level)
- `packages/twenty-front/src`: user-facing application runtime.
Analogy: storefront and clerk stations in a CRM office.
Key file: `modules/app/components/AppRouterProviders.tsx:44-95`.
Common misunderstanding: assuming all state is local React state; much is app-scoped Jotai and provider-driven.

- `packages/twenty-server/src`: backend runtime, API, auth, metadata engine.
Analogy: operations floor where each customer workspace has guarded service lanes.
Key file: `app.module.ts:49-146`.
Misunderstanding: treating all writes as direct ORM saves, while metadata writes run migration validation pipelines.

- `packages/twenty-shared/src`: shared enums/types/utilities used by frontend and backend.
Key files: `types/AppPath.ts:1+`, `types/ViewSortDirection.ts:1+`.
Misunderstanding: thinking frontend/back types are fully codegen-shared only; shared package also enforces cross-layer constants.

## Entry points with annotation
### Frontend `packages/twenty-front/src/index.tsx:1-14`
```tsx
import ReactDOM from 'react-dom/client'; // Browser renderer bootstrap
import { App } from '@/app/components/App'; // App shell entry
// style imports omitted for brevity

const root = ReactDOM.createRoot(
  document.getElementById('root') ?? document.body, // mount target fallback
);

root.render(<App />); // hands off control to provider/routing tree
```

### Backend `packages/twenty-server/src/main.ts:27-96`
```ts
const bootstrap = async () => {
  setPgDateTypeParser(); // normalize PG date parsing globally

  const app = await NestFactory.create<NestExpressApplication>(AppModule, {
    cors: { exposedHeaders: ['WWW-Authenticate'] }, // auth metadata exposure
    bufferLogs: process.env.LOGGER_IS_BUFFER_ENABLED === 'true',
    rawBody: true,
    snapshot: process.env.NODE_ENV === NodeEnvironment.DEVELOPMENT,
  });

  app.use(session(getSessionStorageOptions(twentyConfigService))); // session middleware
  useContainer(app.select(AppModule), { fallbackOnErrors: true }); // DI for class-validator
  app.useGlobalFilters(new UnhandledExceptionFilter()); // shared exception surface

  app.use('/graphql', graphqlUploadExpress({ maxFiles: 10 })); // graphql upload path
  app.use('/metadata', graphqlUploadExpress({ maxFiles: 10 })); // metadata upload path

  generateFrontConfig(); // writes config for frontend consumption
  await app.listen(twentyConfigService.get('NODE_PORT')); // start server
};

void bootstrap();
```

## TypeScript orientation
- Typed user contract: `currentUserState` (`packages/twenty-front/src/modules/auth/states/currentUserState.ts:4-20`).
What it prevents: random user-shape drift between query and consumers.
Without TS: any typo like `canAccessAdminPanel` compiles and fails at runtime.

- Enum-driven routing and workflow constants: `AppPath`, `ViewSortDirection`, `CrudOperationType` (`packages/twenty-shared/src/types/AppPath.ts:1+`, `ViewSortDirection.ts:1+`, `CrudOperationType.ts:1+`).
What it prevents: string literal mismatches and invalid direction values.

## React component anatomies
1. Simple: `Title` (`packages/twenty-front/src/modules/auth/components/Title.tsx:6-35`).
2. Medium: `AuthProvider` (`packages/twenty-front/src/modules/auth/components/AuthProvider.tsx:8-25`).
3. Complex composition: `AppRouterProviders` (`packages/twenty-front/src/modules/app/components/AppRouterProviders.tsx:39-95`).

## Backend route/controller anatomies
1. Dashboard duplicate route (`packages/twenty-server/src/modules/dashboard/controllers/dashboard.controller.ts:11-24`).
2. ViewSort REST controller (`packages/twenty-server/src/engine/metadata-modules/view-sort/controllers/view-sort.controller.ts:38-124`).
3. ViewSort GraphQL resolver (`packages/twenty-server/src/engine/metadata-modules/view-sort/resolvers/view-sort.resolver.ts:23-103`).

## Domain glossary
- Workspace: tenant boundary for users, permissions, metadata (`app.module.ts:118-143`).
- View Sort: ordering metadata attached to a view (`view-sort.entity.ts:35-57`).
- Metadata migration: validated schema/data operation pipeline for metadata objects (`view-sort.service.ts:63-85`, `131-153`).
- Current user workspace: permission and role context in frontend state (`useLoadCurrentUser.ts:75-85`).

## Junior Socratic checkpoint
1. Why does `AppRouterProviders` exist before route rendering?
2. What does `WorkspaceAuthGuard` protect in `ViewSortController`?
3. Why is `ViewSortDirection` enum safer than raw strings?
4. How does `useLoadCurrentUser` distribute one query into multiple states?
5. What breaks if `deletedAt: IsNull()` is removed from `findById`?
6. What is the first backend side effect after app creation?
7. How do frontend and backend agree on route path semantics?

## How to self-grade
Strong answers should mention concrete lines, include failure mode reasoning, and explain one runtime consequence for each concept.
