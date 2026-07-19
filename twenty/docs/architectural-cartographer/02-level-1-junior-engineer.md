# 02 - Level 1: Junior Engineer Guide

## 1.1 Folder-By-Folder Orientation

### Frontend Top-Level Folders

| Folder | Purpose | Twenty-specific analogy | Most important file | Common misunderstanding |
|---|---|---|---|---|
| `packages/twenty-front/src/config` | Holds runtime config values consumed by the frontend. `ApolloProvider` reads `REACT_APP_SERVER_BASE_URL` from config at `packages/twenty-front/src/modules/apollo/components/ApolloProvider.tsx:6`. | The CRM's front desk address book: it tells the browser where the office server lives. | `packages/twenty-front/src/config/index.ts` | Juniors often think config is static build trivia; here it shapes API routing. |
| `packages/twenty-front/src/generated` | Generated app API types and operations. It sits beside generated metadata/admin packages, which means much of the API surface is generated instead of hand-authored. | A printed catalog of every CRM form the frontend knows how to submit. | `packages/twenty-front/src/generated/graphql.ts` | Generated files are read for contracts, not edited casually. |
| `packages/twenty-front/src/generated-admin` | Generated admin-panel GraphQL surface. Admin routes are toggled by `isAdminPageEnabled` in `packages/twenty-front/src/modules/app/components/AppRouter.tsx:12`. | The locked back-office binder for instance admins. | Generated GraphQL entry file in this folder | Juniors may mix admin and workspace APIs; they have separate authorization meaning. |
| `packages/twenty-front/src/generated-metadata` | Generated metadata GraphQL types and documents. `CurrentUser` imports `User` from here at `packages/twenty-front/src/modules/auth/states/currentUserState.ts:2`. | The blueprint binder for workspace fields, views, roles, and object definitions. | `packages/twenty-front/src/generated-metadata/graphql.ts` | "Metadata" is not decorative data; it defines how the app behaves. |
| `packages/twenty-front/src/hooks` | Shared hooks outside feature modules. `usePageChangeEffectNavigateLocation` uses Apollo to validate page layout navigation at `packages/twenty-front/src/hooks/usePageChangeEffectNavigateLocation.ts:61`. | The CRM receptionist's habits used across departments. | `packages/twenty-front/src/hooks/usePageChangeEffectNavigateLocation.ts` | Shared hooks are not automatically generic; read who calls them. |
| `packages/twenty-front/src/loading` | Loading UI and boot-time helpers. It supports app readiness while providers load config, metadata, and auth state. | The "please wait while your workspace opens" lobby screen. | Inspect files under this folder before changing boot loading. | Loading state can hide auth or metadata bugs if treated as mere decoration. |
| `packages/twenty-front/src/locales` | Compiled translations. `App.tsx` activates i18n before rendering at `packages/twenty-front/src/modules/app/components/App.tsx:16`. | The CRM's translated labels for every workspace user. | Locale bundle files | Do not hand-edit generated translation output without knowing the i18n flow. |
| `packages/twenty-front/src/localization` | Runtime localization behavior. `useLoadCurrentUser` activates a user's locale at `packages/twenty-front/src/modules/users/hooks/useLoadCurrentUser.ts:98`. | The per-user language and formatting desk. | `packages/twenty-front/src/modules/users/hooks/useLoadCurrentUser.ts` references it | Locale is tied to user/workspace state, not just a menu choice. |
| `packages/twenty-front/src/modules` | The main feature/module tree. `AppRouterProviders` imports auth, Apollo, metadata, workspace, SSE, UI, and command modules at `packages/twenty-front/src/modules/app/components/AppRouterProviders.tsx:1`. | The departments of the CRM company: sales records, settings, workflows, AI, files, and workspace operations. | `packages/twenty-front/src/modules/app/components/AppRouterProviders.tsx` | The folder is not flat; modules cooperate through providers, atoms, and GraphQL. |
| `packages/twenty-front/src/pages` | Route-level page components. `useCreateAppRouter` lazy-loads `~/pages/object-record/RecordIndexPage` at `packages/twenty-front/src/modules/app/hooks/useCreateAppRouter.tsx:19`. | The named rooms a user can navigate to in the CRM. | `packages/twenty-front/src/modules/app/hooks/useCreateAppRouter.tsx` | Pages are orchestration shells; most reusable behavior lives in modules. |
| `packages/twenty-front/src/testing` | Test utilities, mocks, decorators, and wrappers. Apollo test wrappers appear under `packages/twenty-front/src/testing/jest/getJestMetadataAndApolloMocksWrapper.tsx`. | The practice workspace used to rehearse changes before touching production behavior. | `packages/twenty-front/src/testing/jest/getJestMetadataAndApolloMocksWrapper.tsx` | Test helpers encode real provider assumptions; do not bypass them casually. |
| `packages/twenty-front/src/types` | Ambient and shared frontend-only type declarations. Apollo upload client has a declaration at `packages/twenty-front/src/modules/types/apollo-upload-client.d.ts:1`. | The CRM's local vocabulary cards for libraries TypeScript cannot fully see. | `packages/twenty-front/src/modules/types/apollo-upload-client.d.ts` | Type declarations can hide runtime risk if they overpromise. |
| `packages/twenty-front/src/utils` | General frontend utilities. Apollo error parsing lives in `packages/twenty-front/src/utils/get-error-message-from-apollo-error.util.ts`. | Small desk tools used by multiple CRM departments. | `packages/twenty-front/src/utils/get-error-message-from-apollo-error.util.ts` | Utilities are shared blast-radius code; tiny changes can affect many modules. |

### Backend Top-Level Folders

| Folder | Purpose | Twenty-specific analogy | Most important file | Common misunderstanding |
|---|---|---|---|---|
| `packages/twenty-server/src/command` | CLI/database/operational commands run outside HTTP request handling. Project commands are wired in `packages/twenty-server/project.json:103`. | The maintenance crew that updates the CRM after hours. | `packages/twenty-server/project.json` command target | Commands are production paths too, not throwaway scripts. |
| `packages/twenty-server/src/constants` | Backend constants. Some app-wide settings are imported in `main.ts` from `packages/twenty-server/src/engine/constants/settings` at `packages/twenty-server/src/main.ts:23`. | The office policy manual for limits and defaults. | `packages/twenty-server/src/engine/constants/settings/index.ts` | Constants can have security and performance impact. |
| `packages/twenty-server/src/database` | Database config, TypeORM setup, ClickHouse, migrations, and DB utilities. TypeORM is registered in `packages/twenty-server/src/database/typeorm/typeorm.module.ts:9`. | The storage rooms and filing cabinets behind every CRM record. | `packages/twenty-server/src/database/typeorm/typeorm.module.ts` | Not all data access is direct TypeORM repository code; workspace data sources are layered. |
| `packages/twenty-server/src/engine` | Core platform engine: API layers, guards, metadata modules, middleware, ORM, workspace manager. `AppModule` imports much of it at `packages/twenty-server/src/app.module.ts:16`. | The CRM factory floor that turns workspace metadata into working records, views, and APIs. | `packages/twenty-server/src/app.module.ts` | This is not "misc infrastructure"; it owns the central architecture. |
| `packages/twenty-server/src/filters` | Global and API-specific exception filters. `main.ts` installs `UnhandledExceptionFilter` at `packages/twenty-server/src/main.ts:64`. | The support desk that translates internal failures into user-facing errors. | `packages/twenty-server/src/filters/unhandled-exception.filter.ts` | Error filters are part of the API contract. |
| `packages/twenty-server/src/modules` | Business modules such as workflow and workspace-member. `AppModule` imports `ModulesModule` at `packages/twenty-server/src/app.module.ts:34`. | Product departments built on top of the engine. | `packages/twenty-server/src/modules/modules.module.ts` | Business logic is split between `modules` and engine metadata modules. |
| `packages/twenty-server/src/queue-worker` | Worker process module for async jobs. Root `start` runs `twenty-server:worker` after server startup in `package.json:44`. | The CRM's background processing team for jobs that should not block clicks. | `packages/twenty-server/src/queue-worker/queue-worker.module.ts` | If a feature uses jobs, debugging only HTTP code is incomplete. |
| `packages/twenty-server/src/types` | Shared backend-only type definitions. Request types like `AuthenticatedRequest` are used by REST controllers at `packages/twenty-server/src/engine/api/rest/core/controllers/rest-api-core.controller.ts:19`. | Internal forms passed between backend desks. | `packages/twenty-server/src/engine/api/rest/types/authenticated-request.ts` | Request types often imply middleware has already enriched the request. |
| `packages/twenty-server/src/utils` | Backend utilities and test mocks. Object metadata mocks live at `packages/twenty-server/src/utils/__test__/get-object-metadata-entity.mock.ts`. | Small backend tools and fixture generators. | `packages/twenty-server/src/utils/__test__/get-object-metadata-entity.mock.ts` | Utilities are often close to domain logic in a large codebase. |

### One Level Deeper: Important Frontend Modules

1. `modules/app`: Owns app composition and routing. `App.tsx` provides Jotai and error/i18n shell at `packages/twenty-front/src/modules/app/components/App.tsx:20`, `AppRouter.tsx` connects current-user state to route creation at `packages/twenty-front/src/modules/app/components/AppRouter.tsx:10`, and `useCreateAppRouter.tsx` maps paths to pages at `packages/twenty-front/src/modules/app/hooks/useCreateAppRouter.tsx:109`.

2. `modules/apollo`: Owns API transport, token renewal, GraphQL/REST links, retry, headers, and error handling. `ApolloProvider` points the metadata client at `/metadata` at `packages/twenty-front/src/modules/apollo/components/ApolloProvider.tsx:13`, while `ApolloFactory` builds auth, retry, rest, stream, and upload links at `packages/twenty-front/src/modules/apollo/services/apollo.factory.ts:343`.

3. `modules/auth` plus `modules/users`: Owns current user, workspace, permissions, token pair, and sign-in flows. `currentUserState` narrows generated `User` to the app-needed fields at `packages/twenty-front/src/modules/auth/states/currentUserState.ts:4`, while `useLoadCurrentUser` fans one `currentUser` query into many atoms at `packages/twenty-front/src/modules/users/hooks/useLoadCurrentUser.ts:65`.

### One Level Deeper: Important Backend Engine Areas

1. `engine/api`: Exposes GraphQL, REST, MCP, and API middleware. `AppModule` imports GraphQL API modules and `RestApiModule` at `packages/twenty-server/src/app.module.ts:67`, and `RestApiCoreController` defines wildcard object REST routes at `packages/twenty-server/src/engine/api/rest/core/controllers/rest-api-core.controller.ts:24`.

2. `engine/metadata-modules`: Owns metadata objects such as webhooks, view sorts, view filters, roles, fields, and layout widgets. `ViewSortController` is a REST metadata controller at `packages/twenty-server/src/engine/metadata-modules/view-sort/controllers/view-sort.controller.ts:38`, and `WebhookResolver` is a metadata GraphQL resolver at `packages/twenty-server/src/engine/metadata-modules/webhook/webhook.resolver.ts:24`.

3. `engine/twenty-orm` and `engine/workspace-manager`: Own dynamic workspace data sources and metadata migration execution. `GlobalWorkspaceDataSourceModule` is imported by `AppModule` at `packages/twenty-server/src/app.module.ts:58`, and `ViewSortService` uses workspace migration validation at `packages/twenty-server/src/engine/metadata-modules/view-sort/services/view-sort.service.ts:63`.

## 1.2 Entry Point Walkthroughs

### Frontend Entry: `packages/twenty-front/src/index.tsx`

```tsx
// ReactDOM is the browser renderer. Without it, React components never attach to the DOM.
import ReactDOM from 'react-dom/client';

// App is the first application-owned component. It installs global providers and routing.
import { App } from '@/app/components/App';

// Third-party skeleton styles used by loading placeholders across the app.
import 'react-loading-skeleton/dist/skeleton.css';

// Shared Twenty UI package styles and themes.
import 'twenty-ui/style.css';
import 'twenty-ui/theme-light.css';
import 'twenty-ui/theme-dark.css';

// App-local global CSS.
import './index.css';

// Find the HTML mount node. If root is missing, fall back to document.body so rendering still has a host.
const root = ReactDOM.createRoot(
  document.getElementById('root') ?? document.body,
);

// Hand control to the app shell.
root.render(<App />);
```

First action: create a React root at `packages/twenty-front/src/index.tsx:10`. Removing `ReactDOM` breaks rendering; removing `App` leaves no application; removing style imports breaks global UI styling. Control hands off to `App` at `packages/twenty-front/src/index.tsx:14`.

### Frontend App Shell: `packages/twenty-front/src/modules/app/components/App.tsx`

```tsx
// AppRouter is the first place where URL state becomes a page.
import { AppRouter } from '@/app/components/AppRouter';

// Development-only Apollo logging effect.
import { ApolloDevLogEffect } from '@/debug/components/ApolloDevLogEffect';

// Error boundary catches render-time failures above the router.
import { AppErrorBoundary } from '@/error-handler/components/AppErrorBoundary';
import { AppRootErrorFallback } from '@/error-handler/components/AppRootErrorFallback';

// Exception handling context used by descendants.
import { ExceptionHandlerProvider } from '@/error-handler/components/ExceptionHandlerProvider';

// Instance contexts make globally managed UI elements addressable.
import { SnackBarComponentInstanceContext } from '@/ui/feedback/snack-bar-manager/contexts/SnackBarComponentInstanceContext';
import { ClickOutsideListenerContext } from '@/ui/utilities/pointer-event/contexts/ClickOutsideListenerContext';

// Lingui powers runtime translation.
import { i18n } from '@lingui/core';
import { I18nProvider } from '@lingui/react';

// Jotai store is the app's client-side state container.
import { jotaiStore } from '@/ui/utilities/state/jotai/jotaiStore';
import { Provider as JotaiProvider } from 'jotai';

// Helmet lets routes update document head.
import { HelmetProvider } from 'react-helmet-async';

// Twenty UI icon context.
import { IconsProvider } from 'twenty-ui/display';

// Activate the initial locale before components ask for translations.
import { initialI18nActivate } from '~/utils/i18n/initialI18nActivate';

initialI18nActivate();

export const App = () => {
  return (
    <JotaiProvider store={jotaiStore}>
      <AppErrorBoundary resetOnLocationChange={false} FallbackComponent={AppRootErrorFallback}>
        <I18nProvider i18n={i18n}>
          <ApolloDevLogEffect />
          <SnackBarComponentInstanceContext.Provider value={{ instanceId: 'snack-bar-manager' }}>
            <IconsProvider>
              <ExceptionHandlerProvider>
                <HelmetProvider>
                  <ClickOutsideListenerContext.Provider value={{ excludedClickOutsideId: undefined }}>
                    <AppRouter />
                  </ClickOutsideListenerContext.Provider>
                </HelmetProvider>
              </ExceptionHandlerProvider>
            </IconsProvider>
          </SnackBarComponentInstanceContext.Provider>
        </I18nProvider>
      </AppErrorBoundary>
    </JotaiProvider>
  );
};
```

First action: activate i18n at `packages/twenty-front/src/modules/app/components/App.tsx:16`. Removing Jotai breaks atom-based state; removing the error boundary exposes crashes; removing `I18nProvider` breaks translations; removing `AppRouter` leaves no pages. Control hands off to `AppRouter` at `packages/twenty-front/src/modules/app/components/App.tsx:36`.

### Backend Entry: `packages/twenty-server/src/main.ts`

```ts
// NestFactory creates the HTTP application from AppModule.
import { NestFactory } from '@nestjs/core';
import { type NestExpressApplication } from '@nestjs/platform-express';

// fs is used only when SSL key/cert paths are provided.
import fs from 'fs';

// bytes converts configured max upload size into bytes for upload middleware.
import bytes from 'bytes';

// class-validator needs Nest's DI container for injectable validators.
import { useContainer } from 'class-validator';

// Express session middleware persists session state.
import session from 'express-session';

// GraphQL upload middleware handles multipart file uploads.
import graphqlUploadExpress from 'graphql-upload/graphqlUploadExpress.mjs';

// AppModule is the backend module root.
import { AppModule } from './app.module';

// Instrumentation is imported for side effects before the app runs.
import './instrument';

// bootstrap is the process startup function.
const bootstrap = async () => {
  // Normalize PostgreSQL date parsing before repositories run.
  setPgDateTypeParser();

  // Create a Nest application using the root module.
  const app = await NestFactory.create<NestExpressApplication>(AppModule, {
    cors: { exposedHeaders: ['WWW-Authenticate'] },
    bufferLogs: process.env.LOGGER_IS_BUFFER_ENABLED === 'true',
    rawBody: true,
    snapshot: process.env.NODE_ENV === NodeEnvironment.DEVELOPMENT,
    ...(process.env.SSL_KEY_PATH && process.env.SSL_CERT_PATH
      ? {
          httpsOptions: {
            key: fs.readFileSync(process.env.SSL_KEY_PATH),
            cert: fs.readFileSync(process.env.SSL_CERT_PATH),
          },
        }
      : {}),
  });

  // Pull app services out of Nest DI.
  const logger = app.get(LoggerService);
  const twentyConfigService = app.get(TwentyConfigService);

  // Configure Express proxy behavior from Twenty config.
  app.set('trust proxy', trustProxy);

  // Attach session middleware.
  app.use(session(getSessionStorageOptions(twentyConfigService)));

  // Let class-validator resolve injectable dependencies.
  useContainer(app.select(AppModule), { fallbackOnErrors: true });

  // Use the app logger globally.
  app.useLogger(logger);

  // Convert uncaught exceptions through a consistent filter.
  app.useGlobalFilters(new UnhandledExceptionFilter());

  // Configure body parsing and upload limits.
  app.useBodyParser('json', { limit: settings.storage.maxFileSize });
  app.useBodyParser('urlencoded', { limit: settings.storage.maxFileSize, extended: true });
  app.useBodyParser('text', { type: 'text/plain', limit: '1024kb' });

  // Add upload middleware to GraphQL endpoints.
  app.use('/graphql', graphqlUploadExpress({ maxFieldSize: bytes(settings.storage.maxFileSize)!, maxFiles: 10 }));
  app.use('/metadata', graphqlUploadExpress({ maxFieldSize: bytes(settings.storage.maxFileSize)!, maxFiles: 10 }));

  // Generate frontend config that the browser will consume.
  generateFrontConfig();

  // Start the HTTP server on configured NODE_PORT.
  await app.listen(twentyConfigService.get('NODE_PORT'));
};

void bootstrap();
```

First action: `bootstrap` sets PostgreSQL date parsing at `packages/twenty-server/src/main.ts:28`. Removing `AppModule` prevents all controllers/providers from loading; removing session breaks session-dependent behavior; removing upload middleware breaks GraphQL uploads; removing `generateFrontConfig` risks stale frontend runtime config. Control hands off to `AppModule` when Nest creates the app at `packages/twenty-server/src/main.ts:30`.

## 1.3 TypeScript Orientation

| Pattern | Where | Why it exists | Without TypeScript |
|---|---|---|---|
| Generic state factory | `createAtomState<ValueType>` in `packages/twenty-front/src/modules/ui/utilities/state/jotai/utils/createAtomState.ts:27` | Ensures atom values match their declared type across local/session/cookie storage. | Any caller could write a string into an atom expected to hold a workspace object. |
| Utility type `Pick` | `CurrentUser = Pick<User, ...>` in `packages/twenty-front/src/modules/auth/states/currentUserState.ts:4` | Narrows generated GraphQL `User` to fields the app stores globally. | A component may rely on a huge user object field that was never queried. |
| Nullable union | `CurrentUser | null` in `packages/twenty-front/src/modules/auth/states/currentUserState.ts:18` | Forces callers to handle signed-out or not-yet-loaded user state. | Code may read `currentUser.email` before auth bootstrap completes. |
| Generated GraphQL scalar map | `Scalars` in `packages/twenty-front/src/generated-metadata/graphql.ts:8` | Connects GraphQL schema scalars to TypeScript primitives. | All API responses become unstructured objects. |
| Enum | `ViewSortDirection` in `packages/twenty-front/src/generated-metadata/graphql.ts:5782` and shared version in `packages/twenty-shared/src/types/ViewSortDirection.ts:1` | Prevents invalid sort directions. | UI could submit `"upwards"` and only fail at runtime. |
| Nest DTO return types | `Promise<ViewSortDTO>` in `packages/twenty-server/src/engine/metadata-modules/view-sort/controllers/view-sort.controller.ts:92` | Documents what API methods return and lets services enforce contracts. | A controller could return an entity or partial object accidentally. |
| Typed decorators and request objects | `AuthenticatedRequest` in `packages/twenty-server/src/engine/api/rest/core/controllers/rest-api-core.controller.ts:19` | Signals that guards/middleware add auth and workspace fields. | Route code might assume `workspaceId` exists with no compiler reminder. |
| Interface for factory options | `Options` in `packages/twenty-front/src/modules/apollo/services/apollo.factory.ts:49` | Makes Apollo construction explicit: URI, cache, current workspace, callbacks, app version. | Missing `onUnauthenticatedError` or `currentWorkspace` silently changes runtime behavior. |

## 1.4 Component Anatomy

### Component 1: `App`

Full annotated version appears above. Props: none. State: none locally, but it provides the global `jotaiStore` from `packages/twenty-front/src/modules/ui/utilities/state/jotai/jotaiStore.ts:4`. Render output: global providers and `AppRouter` at `packages/twenty-front/src/modules/app/components/App.tsx:36`. Side effect: `initialI18nActivate()` runs at module load at `packages/twenty-front/src/modules/app/components/App.tsx:16`.

### Component 2: `AppRouter`

```tsx
// Creates the route tree from feature flags and admin access.
import { useCreateAppRouter } from '@/app/hooks/useCreateAppRouter';

// Current user controls admin route availability.
import { currentUserState } from '@/auth/states/currentUserState';

// Local wrapper around Jotai read access.
import { useAtomStateValue } from '@/ui/utilities/state/jotai/hooks/useAtomStateValue';

// React Router renders the router object.
import { RouterProvider } from 'react-router-dom';

export const AppRouter = () => {
  // Kept false while logic function settings are disabled in UI.
  const isFunctionSettingsEnabled = false;

  // Read user from global auth state.
  const currentUser = useAtomStateValue(currentUserState);

  // Only impersonation/full-admin users see admin routes.
  const isAdminPageEnabled =
    (currentUser?.canImpersonate || currentUser?.canAccessFullAdminPanel) ?? false;

  // Hand React Router the generated browser router.
  return (
    <RouterProvider
      router={useCreateAppRouter(isFunctionSettingsEnabled, isAdminPageEnabled)}
    />
  );
};
```

Props: none. State: reads `currentUserState` defined in `packages/twenty-front/src/modules/auth/states/currentUserState.ts:18`. Render output: a `RouterProvider` at `packages/twenty-front/src/modules/app/components/AppRouter.tsx:17`. Side effect: none directly, but route choice changes which lazy pages load.

### Component 3: `ApolloProvider`

```tsx
// ApolloProviderBase makes the Apollo client available to descendant hooks.
import { ApolloProvider as ApolloProviderBase } from '@apollo/client/react';

// useApolloFactory builds the configured ApolloClient instance.
import { useApolloFactory } from '@/apollo/hooks/useApolloFactory';

// Captcha refresh link retries requests that need a fresh captcha.
import { createCaptchaRefreshLink } from '@/apollo/utils/captchaRefreshLink';

// Hook knows how to request a captcha token from the active provider.
import { useRequestFreshCaptchaToken } from '@/captcha/hooks/useRequestFreshCaptchaToken';

// Runtime server base URL.
import { REACT_APP_SERVER_BASE_URL } from '~/config';

export const ApolloProvider = ({ children }: React.PropsWithChildren) => {
  const { requestFreshCaptchaToken } = useRequestFreshCaptchaToken();
  const captchaRefreshLink = createCaptchaRefreshLink(requestFreshCaptchaToken);

  // This provider is specifically configured for metadata GraphQL.
  const apolloClient = useApolloFactory({
    uri: `${REACT_APP_SERVER_BASE_URL}/metadata`,
    devtools: { enabled: process.env.IS_DEBUG_MODE === 'true' },
    extraLinks: [captchaRefreshLink],
  });

  if (process.env.NODE_ENV === 'development') {
    window.__APOLLO_CLIENT__ = apolloClient;
  }

  return <ApolloProviderBase client={apolloClient}>{children}</ApolloProviderBase>;
};
```

Props: `children: React.PropsWithChildren`, passed by `AppRouterProviders` at `packages/twenty-front/src/modules/app/components/AppRouterProviders.tsx:44`. State: none locally; it uses `useApolloFactory`, which reads auth/workspace atoms at `packages/twenty-front/src/modules/apollo/hooks/useApolloFactory.ts:29`. Render output: Apollo context for child components. Side effect: exposes `window.__APOLLO_CLIENT__` in development at `packages/twenty-front/src/modules/apollo/components/ApolloProvider.tsx:20`.

## 1.5 Backend Route Anatomy

### Route 1: Dynamic Object REST API

`RestApiCoreController` is mounted at `@Controller('rest')` and guarded by JWT, workspace auth, and custom permission guards at `packages/twenty-server/src/engine/api/rest/core/controllers/rest-api-core.controller.ts:24`.

```ts
// POST /rest/:objectName creates one dynamic object record.
@Post('*path')
async handleApiPost(@Req() request: AuthenticatedRequest, @Res() res: Response) {
  // request.workspaceId exists because auth/workspace middleware enriched the request.
  this.logger.log(`[REST API] Processing POST request to ${request.path} on workspace ${request.workspaceId}`);

  // Service decides how to parse dynamic object path and write the record.
  const result = await this.restApiCoreService.createOne(request);

  // REST create returns HTTP 201.
  res.status(201).send(result);
}
```

Method/path: `POST /rest/*path` at `packages/twenty-server/src/engine/api/rest/core/controllers/rest-api-core.controller.ts:57`. Middleware/guards: `JwtAuthGuard`, `WorkspaceAuthGuard`, `CustomPermissionGuard` at `packages/twenty-server/src/engine/api/rest/core/controllers/rest-api-core.controller.ts:25`, plus `RestCoreMiddleware` from `packages/twenty-server/src/app.module.ts:140`. Request shape: dynamic path plus request body. Response shape: service result with HTTP 201 at `packages/twenty-server/src/engine/api/rest/core/controllers/rest-api-core.controller.ts:67`. Error handling: `RestApiExceptionFilter` at `packages/twenty-server/src/engine/api/rest/core/controllers/rest-api-core.controller.ts:26`.

### Route 2: View Sort Metadata REST API

```ts
// POST /rest/metadata/viewSorts creates a saved sort for a view.
@Post()
@UseGuards(CreateViewSortPermissionGuard)
async create(
  @Body() input: CreateViewSortInput,
  @AuthWorkspace() { id: workspaceId }: WorkspaceEntity,
): Promise<ViewSortDTO> {
  // Service validates the metadata change and runs a workspace migration.
  return await this.viewSortService.createOne({
    createViewSortInput: input,
    workspaceId,
  });
}
```

Method/path: `POST /rest/metadata/viewSorts` from controller base `packages/twenty-server/src/engine/metadata-modules/view-sort/controllers/view-sort.controller.ts:38` and method decorator at `packages/twenty-server/src/engine/metadata-modules/view-sort/controllers/view-sort.controller.ts:87`. Guards: workspace auth at `packages/twenty-server/src/engine/metadata-modules/view-sort/controllers/view-sort.controller.ts:39` plus create permission guard at `packages/twenty-server/src/engine/metadata-modules/view-sort/controllers/view-sort.controller.ts:88`. Request shape: `CreateViewSortInput` at `packages/twenty-server/src/engine/metadata-modules/view-sort/controllers/view-sort.controller.ts:90`. Response: `Promise<ViewSortDTO>` at `packages/twenty-server/src/engine/metadata-modules/view-sort/controllers/view-sort.controller.ts:92`. Error filters: view sort, flat entity maps, and migration filters at `packages/twenty-server/src/engine/metadata-modules/view-sort/controllers/view-sort.controller.ts:40`.

### Route 3: Webhook Metadata API

```ts
// PATCH /rest/webhooks/:id and /rest/metadata/webhooks/:id update a webhook.
@Patch(':id')
async update(
  @Param('id') id: string,
  @Body() updateWebhookDto: { targetUrl?: string; operations?: string[]; description?: string; secret?: string },
  @AuthWorkspace() workspace: WorkspaceEntity,
): Promise<WebhookDTO> {
  // REST shape is converted into the GraphQL-style UpdateWebhookInput shape.
  const input: UpdateWebhookInput = {
    id,
    update: updateWebhookDto,
  };

  // Service normalizes target URL and runs metadata migration validation.
  return this.webhookService.update(input, workspace.id);
}
```

Method/path: `PATCH /rest/webhooks/:id` and `PATCH /rest/metadata/webhooks/:id` because the controller has two bases at `packages/twenty-server/src/engine/metadata-modules/webhook/controllers/webhook.controller.ts:28`. Guards: `JwtAuthGuard`, `WorkspaceAuthGuard`, and `SettingsPermissionGuard(PermissionFlagType.API_KEYS_AND_WEBHOOKS)` at `packages/twenty-server/src/engine/metadata-modules/webhook/controllers/webhook.controller.ts:29`. Response: `WebhookDTO` at `packages/twenty-server/src/engine/metadata-modules/webhook/controllers/webhook.controller.ts:76`. Service normalizes URLs at `packages/twenty-server/src/engine/metadata-modules/webhook/webhook.service.ts:165`.

## 1.6 Junior Glossary

| Concept | Plain-English definition | Where | Without it | Misconception |
|---|---|---|---|---|
| Nx monorepo | A repo with many packages and shared build/test orchestration. | `package.json:51`, `packages/twenty-front/project.json:37` | Separate repos or manual scripts. | "Monorepo" means one app; here it means many related packages. |
| React provider stack | Components that put services/state into context for descendants. | `packages/twenty-front/src/modules/app/components/AppRouterProviders.tsx:44` | Every page manually wires Apollo, auth, metadata, theme, etc. | Providers are not visual layout. |
| Jotai atom | A small state unit read/written by hooks. | `packages/twenty-front/src/modules/auth/states/currentUserState.ts:18` | Large global object or prop drilling. | Atom state is not server truth; it mirrors loaded server state. |
| Apollo Client | GraphQL/REST client, cache, link chain, and error handling. | `packages/twenty-front/src/modules/apollo/services/apollo.factory.ts:359` | `fetch` calls scattered through components. | Apollo is not only GraphQL here; it also wraps REST links. |
| Generated GraphQL types | TypeScript generated from GraphQL schema/documents. | `packages/twenty-front/src/generated-metadata/graphql.ts:1` | Handwritten request/response types that drift. | Generated types are safe to edit manually. |
| React Router lazy routes | Route components loaded on demand. | `packages/twenty-front/src/modules/app/hooks/useCreateAppRouter.tsx:19` | All pages bundled eagerly. | Lazy loading removes the need to understand the route tree. |
| NestJS module | Backend grouping for providers, controllers, imports. | `packages/twenty-server/src/app.module.ts:49` | Manual Express app wiring. | A module is not the same as a folder. |
| Guard | Nest authorization/authentication gate. | `packages/twenty-server/src/engine/metadata-modules/webhook/controllers/webhook.controller.ts:29` | Handwritten checks inside every route. | If a controller has a guard, method-level permissions no longer matter. |
| Middleware | Request pre-processing before handlers. | `packages/twenty-server/src/app.module.ts:114` | Route handlers parse tokens/workspaces themselves. | Middleware is invisible, so it is optional. It is not optional here. |
| TypeORM repository | Data access wrapper for entities. | `packages/twenty-server/src/engine/metadata-modules/webhook/webhook.service.ts:27` | Raw SQL everywhere. | Repository access is always enough; dynamic workspace records use more layers. |
| Workspace migration | Validated metadata change applied to workspace schema/config. | `packages/twenty-server/src/engine/metadata-modules/view-sort/services/view-sort.service.ts:63` | Direct table/entity mutation. | Metadata changes are just CRUD. |
| Exception filter | Converts exceptions into consistent API responses. | `packages/twenty-server/src/main.ts:64` | Raw stack traces or inconsistent errors. | Error filters are only logging. |
| Path aliases | Imports like `@/auth/...` and `~/utils/...`. | `packages/twenty-front/src/modules/app/components/App.tsx:1` | Long relative imports. | Aliases hide package boundaries; they still map to source roots. |
| Barrel exports | Index files that re-export module APIs. | `packages/twenty-shared/src/index.ts` | Importing deep implementation paths everywhere. | Barrels are always harmless; they can hide dependency cycles. |

## 1.7 Junior Socratic Checkpoint

Use this as an active checkpoint. Do not pause the documentation; answer these in your own words, then compare with the mentor notes.

| Question | Why it matters | Developing answer might say | Strong answer should include |
|---|---|---|---|
| Why does `packages/twenty-front/src/modules/app` exist? | It teaches app composition. | "It has app components." | It owns startup composition: `App`, `AppRouter`, providers, effects, and route creation at `AppRouterProviders.tsx:44`. |
| What is `packages/twenty-front/src/modules/apollo/services/apollo.factory.ts` responsible for? | It is the API boundary. | "It creates Apollo." | It creates Apollo client links, adds auth/workspace/app-version headers, handles token renewal, retries, REST links, upload links, and errors at `apollo.factory.ts:111`. |
| What does `CurrentUser = Pick<User, ...>` protect against? | It explains TS contract narrowing. | "It picks fields." | It keeps global auth state limited to fields the app intentionally stores, avoiding accidental reliance on unqueried `User` fields at `currentUserState.ts:4`. |
| What does `AppRouter` render and why? | It connects state to navigation. | "A router." | It reads `currentUserState`, derives admin access, and passes a browser router to `RouterProvider` at `AppRouter.tsx:17`. |
| What does `POST /rest/*path` do? | It explains dynamic object APIs. | "Creates data." | It creates one dynamic object record, guarded by JWT/workspace/custom permissions, and delegates to `RestApiCoreService.createOne` at `rest-api-core.controller.ts:65`. |
| What breaks if `AppModule` is deleted? | It tests backend composition. | "The backend won't start." | Nest cannot load GraphQL, REST, ORM, core modules, business modules, middleware, or static frontend serving because `AppModule` imports them at `app.module.ts:49`. |
| How are frontend and backend connected? | It tests full-stack thinking. | "Apollo calls the backend." | The frontend config supplies server base URL, Apollo adds auth/workspace headers in `apollo.factory.ts:111`, and backend middleware hydrates request context in `app.module.ts:114`. |

