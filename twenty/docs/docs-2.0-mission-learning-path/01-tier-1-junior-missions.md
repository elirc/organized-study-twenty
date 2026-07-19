# Tier 1 - Junior Missions

## Mission 1: The App's Heartbeat

**Tier:** Junior  
**Time Estimate:** 45 minutes  
**Goal:** You will understand how the frontend and backend boot, and where the app first becomes "alive" on each side.

**The Concept:**

Bootstrapping is the moment a program creates its first runtime shape. In this CRM, the frontend bootstrap is like opening the workspace front door: React attaches to the browser and starts loading providers, routes, user state, and metadata. The backend bootstrap is like turning on the workspace operations center: Nest loads modules, middleware, sessions, upload handling, GraphQL, REST, and background-ready services before accepting requests.

**Design Intent (Before You Read the Code):**

1. Files involved: `packages/twenty-front/src/index.tsx:1-14`, `packages/twenty-front/src/modules/app/components/App.tsx:1-46`, `packages/twenty-server/src/main.ts:1-96`, `packages/twenty-server/src/app.module.ts:49-146`.
2. `index.tsx` owns DOM-to-React handoff. `App.tsx` owns the first frontend provider shell. `main.ts` owns Nest process startup. `app.module.ts` owns backend module and middleware composition.
3. Existing patterns: small entry file, provider composition, Nest root module, global middleware before routes.
4. Poor implementation would cause blank screens, missing global state, requests without workspace context, broken uploads, inconsistent errors, or a server that listens before dependencies are ready.

**Find It In The Code:**

Open these ranges:

- `packages/twenty-front/src/index.tsx:1-14`
- `packages/twenty-front/src/modules/app/components/App.tsx:16-44`
- `packages/twenty-server/src/main.ts:27-96`
- `packages/twenty-server/src/app.module.ts:49-146`

```tsx
// packages/twenty-front/src/index.tsx:1-14
// ReactDOM is the browser renderer; without it, JSX stays inert JavaScript objects.
import ReactDOM from 'react-dom/client';

// App is the first application-owned component.
import { App } from '@/app/components/App';

// Global styles load before the app renders so shared UI components look correct.
import 'react-loading-skeleton/dist/skeleton.css';
import 'twenty-ui/style.css';
import 'twenty-ui/theme-light.css';
import 'twenty-ui/theme-dark.css';
import './index.css';

// This is the browser bootstrap: find a DOM node and let React own it.
const root = ReactDOM.createRoot(
  document.getElementById('root') ?? document.body,
);

// Control now moves from plain browser DOM into the React app.
root.render(<App />);
```

```tsx
// packages/twenty-front/src/modules/app/components/App.tsx:16-44
// Translations must be activated before UI text renders.
initialI18nActivate();

export const App = () => {
  return (
    // JotaiProvider gives every descendant access to the app-wide atom store.
    <JotaiProvider store={jotaiStore}>
      // AppErrorBoundary prevents one render failure from taking down the entire shell.
      <AppErrorBoundary resetOnLocationChange={false} FallbackComponent={AppRootErrorFallback}>
        // I18nProvider makes Lingui translations available to all routes.
        <I18nProvider i18n={i18n}>
          <ApolloDevLogEffect />
          <SnackBarComponentInstanceContext.Provider value={{ instanceId: 'snack-bar-manager' }}>
            <IconsProvider>
              <ExceptionHandlerProvider>
                <HelmetProvider>
                  <ClickOutsideListenerContext.Provider value={{ excludedClickOutsideId: undefined }}>
                    // The app shell hands control to URL-based routing here.
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

```ts
// packages/twenty-server/src/main.ts:27-96
// Backend bootstrap is async because the server must create modules and read config first.
const bootstrap = async () => {
  // Normalize PostgreSQL date parsing before repositories start reading dates.
  setPgDateTypeParser();

  // Create the Nest app from the root module.
  const app = await NestFactory.create<NestExpressApplication>(AppModule, {
    cors: { exposedHeaders: ['WWW-Authenticate'] },
    bufferLogs: process.env.LOGGER_IS_BUFFER_ENABLED === 'true',
    rawBody: true,
    snapshot: process.env.NODE_ENV === NodeEnvironment.DEVELOPMENT,
  });

  // Resolve logger/config from Nest's dependency injection container.
  const logger = app.get(LoggerService);
  const twentyConfigService = app.get(TwentyConfigService);

  // Sessions, validators, logger, exception filters, body parsers, and uploads are installed before listen.
  app.use(session(getSessionStorageOptions(twentyConfigService)));
  useContainer(app.select(AppModule), { fallbackOnErrors: true });
  app.useLogger(logger);
  app.useGlobalFilters(new UnhandledExceptionFilter());
  app.useBodyParser('json', { limit: settings.storage.maxFileSize });
  app.use('/graphql', graphqlUploadExpress({ maxFieldSize: bytes(settings.storage.maxFileSize)!, maxFiles: 10 }));
  app.use('/metadata', graphqlUploadExpress({ maxFieldSize: bytes(settings.storage.maxFileSize)!, maxFiles: 10 }));

  // The backend writes frontend runtime config before accepting traffic.
  generateFrontConfig();

  // This is when the backend becomes reachable over HTTP.
  await app.listen(twentyConfigService.get('NODE_PORT'));
};

// Fire the bootstrap promise without awaiting at top level.
void bootstrap();
```

```ts
// packages/twenty-server/src/app.module.ts:49-146
@Module({
  imports: [
    // GraphQL endpoint and schema config.
    GraphQLModule.forRootAsync<YogaDriverConfig>({
      driver: YogaDriver,
      imports: [GraphQLConfigModule, MetricsModule, DataloaderModule],
      useClass: GraphQLConfigService,
    }),
    // ORM and workspace data source layers.
    TwentyORMModule,
    GlobalWorkspaceDataSourceModule,
    // Product and platform modules.
    CoreEngineModule,
    ModulesModule,
    CoreGraphQLApiModule,
    MetadataGraphQLApiModule,
    AdminPanelGraphQLApiModule,
    RestApiModule,
    McpModule,
  ],
})
export class AppModule {
  configure(consumer: MiddlewareConsumer) {
    // GraphQL requests get token hydration and workspace auth context before resolvers run.
    consumer
      .apply(GraphQLHydrateRequestFromTokenMiddleware, WorkspaceAuthContextMiddleware)
      .forRoutes({ path: 'graphql', method: RequestMethod.ALL });

    // Metadata and admin GraphQL get the same request-context setup.
    consumer
      .apply(GraphQLHydrateRequestFromTokenMiddleware, WorkspaceAuthContextMiddleware)
      .forRoutes({ path: 'metadata', method: RequestMethod.ALL });

    // REST object routes get REST-specific parsing plus workspace context.
    for (const method of MIGRATED_REST_METHODS) {
      consumer
        .apply(RestCoreMiddleware, WorkspaceAuthContextMiddleware)
        .forRoutes({ path: 'rest/*path', method });
    }
  }
}
```

**The "Aha" Moment:**

Bootstrapping is not "starting the app"; it is installing the assumptions every later file depends on.

**Socratic Checkpoint:**

Answer before continuing:

1. What is the first line where the frontend becomes a React app?
2. Why does `App.tsx` wrap the router in providers instead of letting each page import what it needs?
3. What must happen before `app.listen` on the backend?
4. How does the backend know which middleware should run before `/graphql` requests?
5. How do frontend and backend know about each other?

Self-grade rubric: strong answers cite `index.tsx:10-14`, `App.tsx:20-44`, `main.ts:27-96`, `app.module.ts:114-146`, and the frontend config/server base URL path used by Apollo in `packages/twenty-front/src/modules/apollo/components/ApolloProvider.tsx:13-17`.

**Connects To:**

- Mission 2, because folder structure makes sense after boot flow.
- Mission 8, because routing is the frontend skeleton after bootstrap.

## Mission 2: The Folder Mental Map

**Tier:** Junior  
**Time Estimate:** 40 minutes  
**Goal:** You will be able to explain what the main frontend and backend folders own.

**The Concept:**

A large repo is a workspace with departments. In Twenty, frontend folders are the customer-facing CRM office: pages are rooms, modules are departments, generated folders are printed API contracts, and testing is the rehearsal space. Backend folders are operations: engine runs the platform, modules run product departments, database stores records, queue-worker handles background work.

**Design Intent (Before You Read the Code):**

1. Files involved: folder roots under `packages/twenty-front/src` and `packages/twenty-server/src`.
2. Each top-level folder owns a different kind of responsibility: entry, feature modules, generated contracts, runtime config, tests, backend engine, database, commands, and workers.
3. Existing pattern: frontend features mostly live under `modules`; backend platform code mostly lives under `engine`; business modules live under `modules`.
4. Poor folder ownership leads to features reaching across boundaries, generated files being edited manually, and debugging starting in the wrong place.

**Find It In The Code:**

Open:

- `packages/twenty-front/src:1-13` as folder listing from `Get-ChildItem`
- `packages/twenty-front/src/modules:1-51` as module listing
- `packages/twenty-server/src:1-9` as backend folder listing
- `packages/twenty-server/src/engine:1-18` as engine folder listing

```text
// Frontend top-level mental map.
config              // Runtime settings used by API/config providers.
generated           // Generated GraphQL/API contracts; read, usually do not hand-edit.
generated-admin     // Admin GraphQL contract area.
generated-metadata  // Metadata GraphQL contract area, central for workspace-driven behavior.
hooks               // Shared hooks not owned by one feature department.
loading             // Boot/loading UI while user/metadata/config are not ready.
locales             // Compiled translation data.
localization        // Runtime formatting and language behavior.
modules             // CRM feature departments: auth, views, records, workspace, settings, workflow.
pages               // Route-level rooms the user can visit.
testing             // Wrappers, mocks, decorators, and test helpers.
types               // Frontend-only type declarations.
utils               // Small shared helpers.
```

```text
// Backend top-level mental map.
command       // Operational commands and scripts.
constants     // Backend constants and settings.
database      // TypeORM, migrations, ClickHouse, setup.
engine        // Platform engine: API, guards, middleware, metadata, ORM, workspace manager.
filters       // Exception filters that translate failures.
modules       // Product/business modules built on top of the engine.
queue-worker  // Background worker process.
types         // Backend type helpers.
utils         // Shared backend utilities and test fixtures.
```

**The "Aha" Moment:**

A folder map is a debugging map: the faster you know ownership, the faster you know where not to look.

**Socratic Checkpoint:**

1. Where would you look first for a route-level React page?
2. Where would you look first for Apollo request behavior?
3. Where would you look first for backend auth guards?
4. Which folders are likely generated contracts?
5. Why is `engine` not a dumping ground?

Self-grade rubric: strong answers mention `pages`, `modules/apollo`, `engine/guards`, `generated*`, and the fact that `engine` owns platform concerns imported by `AppModule` at `packages/twenty-server/src/app.module.ts:16-34`.

**Connects To:**

- Mission 3, because generated/shared types live in known folders.
- Mission 5, because backend routes live under engine/modules controllers.

## Mission 3: TypeScript Is a Contract

**Tier:** Junior  
**Time Estimate:** 35 minutes  
**Goal:** You will understand TypeScript as a runtime-risk reduction tool, not just syntax.

**The Concept:**

In this CRM, a type is like a workspace form template: it says which fields are allowed before someone tries to file the form. `CurrentUser` is a good first contract because auth state controls route access, workspace identity, permissions, and personalization.

**Design Intent (Before You Read the Code):**

1. Files involved: `packages/twenty-front/src/modules/auth/states/currentUserState.ts:1-21`, `packages/twenty-front/src/generated-metadata/graphql.ts:5533-5561`.
2. `currentUserState.ts` owns the narrowed current-user state stored in Jotai.
3. Pattern: generated GraphQL type -> narrowed app state type -> atom state.
4. Poor implementation would let components assume fields exist when the query never loaded them, causing runtime `undefined` bugs.

**Find It In The Code:**

```ts
// packages/twenty-front/src/modules/auth/states/currentUserState.ts:1-21
// createAtomState wraps Jotai atom creation in the app's standard state shape.
import { createAtomState } from '@/ui/utilities/state/jotai/utils/createAtomState';

// User comes from generated GraphQL metadata types, so it mirrors the API schema.
import { type User } from '~/generated-metadata/graphql';

// Pick means: global current-user state may only store these fields from User.
export type CurrentUser = Pick<
  User,
  | 'id'
  | 'email'
  | 'supportUserHash'
  | 'canAccessFullAdminPanel'
  | 'canImpersonate'
  | 'onboardingStatus'
  | 'userVars'
  | 'firstName'
  | 'lastName'
  | 'hasPassword'
>;

// The user may be absent before loading or after sign-out, so null is part of the contract.
export const currentUserState = createAtomState<CurrentUser | null>({
  key: 'currentUserState',
  defaultValue: null,
});
```

Runtime break without it: a component could treat `currentUser` as always present and read `currentUser.email` during boot. `AppRouter` avoids that with optional access at `packages/twenty-front/src/modules/app/components/AppRouter.tsx:10-14`.

**The "Aha" Moment:**

TypeScript is valuable when it makes impossible states visible before users click into them.

**Socratic Checkpoint:**

1. Why is `CurrentUser` a `Pick<User, ...>` instead of `User`?
2. Why is `null` part of the type?
3. What runtime bug does optional access in `AppRouter.tsx:12-14` avoid?
4. What file would need to change if the app needed `currentUser.avatarUrl` globally?
5. What is the difference between generated API type and app state type?

Self-grade rubric: strong answers distinguish API schema from locally stored state and mention `useLoadCurrentUser` writing user state at `packages/twenty-front/src/modules/users/hooks/useLoadCurrentUser.ts:65-84`.

**Connects To:**

- Mission 6, because props are also typed contracts.
- Mission 9, because this type lives in global state for a reason.

## Mission 4: Your First React Component

**Tier:** Junior  
**Time Estimate:** 30 minutes  
**Goal:** You will understand JSX, props, conditional rendering, and typed styling through one small component.

**The Concept:**

A React component is a reusable CRM UI stamp. `Title` is like the standard heading printed at the top of auth forms: it accepts text, optionally animates it, and optionally removes top margin.

**Design Intent (Before You Read the Code):**

1. File involved: `packages/twenty-front/src/modules/auth/components/Title.tsx:1-35`.
2. It owns one piece of auth UI typography.
3. Pattern: typed props, styled component, defaults, conditional render.
4. Poor implementation would make auth screens visually inconsistent or make animation/margins unpredictable.

**Find It In The Code:**

```tsx
// packages/twenty-front/src/modules/auth/components/Title.tsx:1-35
// Linaria creates CSS at build time from this styled component.
import { styled } from '@linaria/react';

// React is needed for PropsWithChildren typing and JSX support.
import React from 'react';

// Theme variables keep this title aligned with Twenty UI.
import { themeCssVariables } from 'twenty-ui/theme-constants';

// AnimatedEaseIn is an optional wrapper around children.
import { AnimatedEaseIn } from 'twenty-ui/utilities';

// PropsWithChildren means this component accepts nested JSX/text.
type TitleProps = React.PropsWithChildren & {
  animate?: boolean;
  noMarginTop?: boolean;
};

// StyledTitle accepts only noMarginTop because styling only needs that prop.
const StyledTitle = styled.div<Pick<TitleProps, 'noMarginTop'>>`
  color: ${themeCssVariables.font.color.primary};
  font-size: ${themeCssVariables.font.size.xl};
  font-weight: ${themeCssVariables.font.weight.semiBold};
  margin-bottom: ${themeCssVariables.spacing[4]};
  margin-top: ${({ noMarginTop }) =>
    !noMarginTop ? themeCssVariables.spacing[4] : '0'};
  text-align: center;
`;

// Defaults make callers simple: animation and margin override are opt-in.
export const Title = ({
  children,
  animate = false,
  noMarginTop = false,
}: TitleProps) => {
  if (animate) {
    return (
      <StyledTitle noMarginTop={noMarginTop}>
        <AnimatedEaseIn>{children}</AnimatedEaseIn>
      </StyledTitle>
    );
  }

  return <StyledTitle noMarginTop={noMarginTop}>{children}</StyledTitle>;
};
```

**The "Aha" Moment:**

Props are the component's public API, and JSX is just the component returning the UI shape for the current props.

**Socratic Checkpoint:**

1. What does `children` represent here?
2. Why are `animate` and `noMarginTop` optional?
3. Why does `StyledTitle` use `Pick<TitleProps, 'noMarginTop'>`?
4. What changes in the rendered output when `animate` is true?
5. What would break visually if theme variables were replaced with hard-coded values?

Self-grade rubric: strong answers describe public API, default props, conditional render, and theme consistency.

**Connects To:**

- Mission 6, because parent-child prop contracts are the next step.
- Mission 16, because composition scales this same idea.

## Mission 5: Your First Node.js Route

**Tier:** Junior  
**Time Estimate:** 35 minutes  
**Goal:** You will understand a simple Nest route, guards, request input, service delegation, and response type.

**The Concept:**

A backend route is a staffed service desk. In Twenty's CRM, duplicating a dashboard is a desk where the user presents a dashboard ID, security checks confirm workspace access, and a service performs the actual copy.

**Design Intent (Before You Read the Code):**

1. File involved: `packages/twenty-server/src/modules/dashboard/controllers/dashboard.controller.ts:1-25`.
2. The controller owns the HTTP boundary for dashboard duplication.
3. Pattern: controller decorator, guards, filter, service injection, typed return.
4. Poor implementation could duplicate dashboards across workspaces, leak errors, or return an untyped shape.

**Find It In The Code:**

```ts
// packages/twenty-server/src/modules/dashboard/controllers/dashboard.controller.ts:1-25
// Nest route decorators describe HTTP behavior.
import { Controller, Param, Post, UseFilters, UseGuards } from '@nestjs/common';

// Workspace auth context was created earlier by middleware.
import { getWorkspaceAuthContext } from 'src/engine/core-modules/auth/storage/workspace-auth-context.storage';

// Guards protect the route before business logic runs.
import { JwtAuthGuard } from 'src/engine/guards/jwt-auth.guard';
import { NoPermissionGuard } from 'src/engine/guards/no-permission.guard';
import { WorkspaceAuthGuard } from 'src/engine/guards/workspace-auth.guard';

// DTO says what the route returns.
import { DuplicatedDashboardDTO } from 'src/modules/dashboard/dtos/duplicated-dashboard.dto';

// Filter translates dashboard exceptions into API responses.
import { DashboardRestApiExceptionFilter } from 'src/modules/dashboard/filters/dashboard-rest-api-exception.filter';

// Service owns the actual duplication work.
import { DashboardDuplicationService } from 'src/modules/dashboard/services/dashboard-duplication.service';

@Controller('rest/dashboards')
@UseGuards(JwtAuthGuard, WorkspaceAuthGuard, NoPermissionGuard)
@UseFilters(DashboardRestApiExceptionFilter)
export class DashboardController {
  constructor(
    private readonly dashboardDuplicationService: DashboardDuplicationService,
  ) {}

  // POST is used because duplicating creates a new dashboard derived from an existing one.
  @Post(':id/duplicate')
  async duplicate(@Param('id') id: string): Promise<DuplicatedDashboardDTO> {
    // The route reads workspace/user context after guards and middleware have prepared it.
    const authContext = getWorkspaceAuthContext();

    // Controller delegates to service and returns the typed DTO.
    return this.dashboardDuplicationService.duplicateDashboard(id, authContext);
  }
}
```

Frontend connection: the frontend calls backend APIs through Apollo/REST links built in `packages/twenty-front/src/modules/apollo/services/apollo.factory.ts:98-109` and headers added at `packages/twenty-front/src/modules/apollo/services/apollo.factory.ts:111-140`.

**The "Aha" Moment:**

A controller should be thin: validate access, read request inputs, delegate business work, and return a typed response.

**Socratic Checkpoint:**

1. Why is dashboard duplication a `POST`?
2. What do guards do before the controller method runs?
3. Why is `getWorkspaceAuthContext` needed?
4. What does the DTO protect?
5. What should not be inside this controller method?

Self-grade rubric: strong answers separate HTTP boundary from business logic and mention `DashboardDuplicationService`.

**Connects To:**

- Mission 12, because this route is one entry in the API surface.
- Mission 13, because middleware prepares context before guards/controllers.

## Mission 6: Props Are a Typed Contract

**Tier:** Junior  
**Time Estimate:** 30 minutes  
**Goal:** You will trace a typed prop from caller to child component.

**The Concept:**

Typed props are like passing a CRM record card between departments: the receiving department should know exactly which fields are present. `AppRouter` passes route-building flags to `useCreateAppRouter`, and settings routes receive admin/function flags.

**Design Intent (Before You Read the Code):**

1. Files involved: `packages/twenty-front/src/modules/app/components/AppRouter.tsx:6-20`, `packages/twenty-front/src/modules/app/hooks/useCreateAppRouter.tsx:109-244`.
2. `AppRouter` derives state. `useCreateAppRouter` builds routes. `SettingsRoutes` receives the flags.
3. Pattern: derive boolean from state, pass typed function params, pass props into child route component.
4. Poor implementation would expose admin/settings pages to the wrong users or hide pages from valid users.

**Find It In The Code:**

```tsx
// packages/twenty-front/src/modules/app/components/AppRouter.tsx:6-20
export const AppRouter = () => {
  // Feature flag currently hard-coded off.
  const isFunctionSettingsEnabled = false;

  // currentUser comes from global Jotai auth state.
  const currentUser = useAtomStateValue(currentUserState);

  // Route access is derived from user capabilities.
  const isAdminPageEnabled =
    (currentUser?.canImpersonate || currentUser?.canAccessFullAdminPanel) ?? false;

  // The booleans become the route factory's typed inputs.
  return (
    <RouterProvider
      router={useCreateAppRouter(isFunctionSettingsEnabled, isAdminPageEnabled)}
    />
  );
};
```

```tsx
// packages/twenty-front/src/modules/app/hooks/useCreateAppRouter.tsx:109-244
export const useCreateAppRouter = (
  // These booleans are the contract from AppRouter into route construction.
  isFunctionSettingsEnabled?: boolean,
  isAdminPageEnabled?: boolean,
) =>
  createBrowserRouter(
    createRoutesFromElements(
      <Route element={<AppRouterProviders />}>
        <Route element={<DefaultLayout />}>
          <Route
            path={AppPath.SettingsCatchAll}
            element={
              // SettingsRoutes receives the flags and decides which settings screens are available.
              <SettingsRoutes
                isFunctionSettingsEnabled={isFunctionSettingsEnabled}
                isAdminPageEnabled={isAdminPageEnabled}
              />
            }
          />
        </Route>
      </Route>,
    ),
  );
```

**The "Aha" Moment:**

Typed props turn "I hope this child gets what it needs" into a visible boundary.

**Socratic Checkpoint:**

1. Where does `isAdminPageEnabled` originate?
2. Why is it derived instead of stored separately?
3. Which child receives it?
4. What bug could happen if this prop defaulted to true?
5. Why is this a contract?

Self-grade rubric: strong answers cite `AppRouter.tsx:10-18` and `useCreateAppRouter.tsx:238-244`.

**Connects To:**

- Mission 8, because props influence route availability.
- Mission 9, because this shows derived state ownership.

## Mission 7: Following Data Into the App

**Tier:** Junior  
**Time Estimate:** 45 minutes  
**Goal:** You will trace current-user data from backend request to frontend state.

**The Concept:**

Data entering Twenty is like a workspace badge being scanned at the front desk. The backend confirms who you are; the frontend then fills current user, workspace, members, permissions, locale, and routing decisions.

**Design Intent (Before You Read the Code):**

1. Files involved: `packages/twenty-front/src/modules/users/hooks/useLoadCurrentUser.ts:24-153`, `packages/twenty-front/src/modules/users/graphql/queries/getCurrentUser.ts:1-12`, `packages/twenty-front/src/modules/auth/states/currentUserState.ts:18-21`.
2. The query file defines what data to ask for. The hook runs the query and populates state. The atom stores current user.
3. Pattern: generated GraphQL document -> Apollo query -> Jotai state fan-out.
4. Poor implementation would leave partial auth state, wrong locale, wrong workspace domain, or broken permissions.

**Find It In The Code:**

```ts
// packages/twenty-front/src/modules/users/hooks/useLoadCurrentUser.ts:45-65
// Apollo client comes from the ApolloProvider above this hook in the tree.
const client = useApolloClient();

const loadCurrentUser = useCallback(async () => {
  // This is the first explicit current-user request.
  const currentUserResult = await client.query({
    query: GetCurrentUserDocument,
    fetchPolicy: 'network-only',
  });

  // A GraphQL result error becomes a thrown JS error.
  if (isDefined(currentUserResult.error)) {
    throw new Error(currentUserResult.error.message);
  }

  // The hook expects the API to return currentUser.
  const user = currentUserResult.data?.currentUser;

  if (!isDefined(user)) {
    throw new Error('No current user result');
  }

  // This writes the main current-user atom.
  setCurrentUser(user);
```

```ts
// packages/twenty-front/src/modules/users/hooks/useLoadCurrentUser.ts:67-128
// The same response fans out into related state slices.
if (isDefined(user.workspaceMembers)) {
  setCurrentWorkspaceMembers(user.workspaceMembers);
}

if (isDefined(user.availableWorkspaces)) {
  setAvailableWorkspaces(user.availableWorkspaces);
}

if (isDefined(user.currentUserWorkspace)) {
  setCurrentUserWorkspace({
    permissionFlags: user.currentUserWorkspace.permissionFlags ?? [],
    twoFactorAuthenticationMethodSummary:
      user.currentUserWorkspace.twoFactorAuthenticationMethodSummary ?? [],
    objectsPermissions:
      (user.currentUserWorkspace.objectsPermissions as Array<
        ObjectPermissions & { objectMetadataId: string }
      >) ?? [],
  });
}

// Locale and formatting are initialized from workspace member.
if (isDefined(user.workspaceMember)) {
  setCurrentWorkspaceMember(workspaceMember);
  initializeFormatPreferences(workspaceMember);
  dynamicActivate((workspaceMember.locale as keyof typeof APP_LOCALES) ?? SOURCE_LOCALE);
}
```

Flow diagram:

```text
Component/effect calls loadCurrentUser
  -> Apollo client.query(GetCurrentUserDocument)
  -> backend /metadata or /graphql currentUser resolver
  -> generated result data.currentUser
  -> setCurrentUser
  -> set workspace/member/permissions/locale atoms
  -> AppRouter and feature modules read those atoms
  -> UI routes and permissions adapt
```

**The "Aha" Moment:**

The first data fetch is not just data loading; it establishes the user's entire workspace identity.

**Socratic Checkpoint:**

1. Why is `fetchPolicy: 'network-only'` reasonable here?
2. Why does one API response update many atoms?
3. What could break if `currentUserWorkspace` were not set?
4. Why does locale activation happen in this hook?
5. Where would you debug a user with correct login but wrong workspace?

Self-grade rubric: strong answers cite `useLoadCurrentUser.ts:48-65`, `useLoadCurrentUser.ts:75-84`, and `useLoadCurrentUser.ts:94-127`.

**Connects To:**

- Mission 9, because it teaches state ownership.
- Mission 14, because it is a full-stack data trace template.

## Mission 8: Navigation Is the App's Skeleton

**Tier:** Junior  
**Time Estimate:** 45 minutes  
**Goal:** You will understand how URLs become layouts and pages.

**The Concept:**

Navigation is the CRM building floor plan. Layout routes are hallways, protected/provider routes are security doors, and leaf routes are rooms like record pages, settings, auth, or onboarding.

**Design Intent (Before You Read the Code):**

1. Files involved: `packages/twenty-front/src/modules/app/components/AppRouter.tsx:6-20`, `packages/twenty-front/src/modules/app/hooks/useCreateAppRouter.tsx:109-260`.
2. `AppRouter` creates a router. `useCreateAppRouter` maps paths to route elements.
3. Pattern: lazy route imports, shared providers at top route, layout route nesting, leaf pages.
4. Poor implementation would create broken deep links, missing providers, wrong layout, or unauthorized pages.

**Find It In The Code:**

```tsx
// packages/twenty-front/src/modules/app/hooks/useCreateAppRouter.tsx:109-121
export const useCreateAppRouter = (
  isFunctionSettingsEnabled?: boolean,
  isAdminPageEnabled?: boolean,
) =>
  createBrowserRouter(
    createRoutesFromElements(
      // Top route installs AppRouterProviders for all children.
      <Route
        element={<AppRouterProviders />}
        loader={async () => Promise.resolve(null)}
      >
        // DefaultLayout wraps normal app pages.
        <Route element={<DefaultLayout />}>
```

```tsx
// packages/twenty-front/src/modules/app/hooks/useCreateAppRouter.tsx:121-245
// Auth and onboarding routes are leaf routes under DefaultLayout.
<Route path={AppPath.SignInUp} element={<LazyRoute fallback={null}><SignInUp /></LazyRoute>} />
<Route path={AppPath.CreateWorkspace} element={<LazyRoute fallback={null}><CreateWorkspace /></LazyRoute>} />

// CRM object routes are leaf routes that load record pages.
<Route path={AppPath.RecordIndexPage} element={<LazyRoute><RecordIndexPage /></LazyRoute>} />
<Route path={AppPath.RecordShowPage} element={<LazyRoute><RecordShowPage /></LazyRoute>} />

// SettingsRoutes receives feature/admin flags.
<Route
  path={AppPath.SettingsCatchAll}
  element={
    <SettingsRoutes
      isFunctionSettingsEnabled={isFunctionSettingsEnabled}
      isAdminPageEnabled={isAdminPageEnabled}
    />
  }
/>
```

```tsx
// packages/twenty-front/src/modules/app/hooks/useCreateAppRouter.tsx:255-260
// BlankLayout wraps OAuth/authorize style pages that should not use the normal app layout.
<Route element={<BlankLayout />}>
  <Route
    path={AppPath.Authorize}
    element={<LazyRoute><Authorize /></LazyRoute>}
  />
</Route>
```

Direct URL flow:

```text
Browser URL
  -> index.tsx renders App
  -> AppRouter renders RouterProvider
  -> createBrowserRouter matches path
  -> AppRouterProviders mount
  -> layout route renders
  -> leaf route lazy-loads page
```

**The "Aha" Moment:**

Routing is not just URLs; it decides which runtime providers, layouts, permissions, and page modules exist for a screen.

**Socratic Checkpoint:**

1. What is a layout route in this code?
2. Why does `AppRouterProviders` sit above `DefaultLayout`?
3. What makes `RecordIndexPage` lazy?
4. Why does `Authorize` use `BlankLayout`?
5. What happens when a user directly opens a record URL?

Self-grade rubric: strong answers cite `useCreateAppRouter.tsx:115-121`, `useCreateAppRouter.tsx:214-225`, and `useCreateAppRouter.tsx:255-260`.

**Connects To:**

- Mission 9, because routes consume state.
- Mission 16, because layout and provider composition are advanced React design.

