# Tier 2 - Mid-Level Missions

## Mission 9: State Has a Home - and a Reason

**Tier:** Mid-Level  
**Time Estimate:** 50 minutes  
**Goal:** You will classify state by owner: local UI, global client state, server cache, route state, and backend-persisted state.

**The Concept:**

State ownership asks: which part of the CRM is responsible for remembering this fact? A field input can remember local typing, the workspace shell remembers the current user, Apollo remembers server responses, the URL remembers navigation, and the backend remembers durable workspace records.

**Design Intent (Before You Read the Code):**

1. Files involved: `packages/twenty-front/src/modules/ui/utilities/state/jotai/utils/createAtomState.ts:27-79`, `packages/twenty-front/src/modules/auth/states/currentUserState.ts:4-21`, `packages/twenty-front/src/modules/apollo/services/apollo.factory.ts:359-365`, `packages/twenty-front/src/modules/app/hooks/useCreateAppRouter.tsx:109-260`.
2. Jotai atoms own app-global client state. Apollo owns server cache. React Router owns URL-to-page state.
3. Pattern: persistent global facts become atoms; server facts flow through Apollo; route facts flow through router; short-lived UI facts stay local.
4. Poor state ownership creates stale UI, duplicated sources of truth, cache bugs, and hard-to-debug route behavior.

**Find It In The Code:**

```ts
// packages/twenty-front/src/modules/ui/utilities/state/jotai/utils/createAtomState.ts:27-79
// This factory makes state ownership explicit: a state has a key, default value, storage choice, and atom.
export const createAtomState = <ValueType>({
  key,
  defaultValue,
  useLocalStorage = false,
  useSessionStorage = false,
  useCookieStorage,
}: {
  key: string;
  defaultValue: ValueType;
  useLocalStorage?: boolean;
  useSessionStorage?: boolean;
  useCookieStorage?: CookieStorageConfig<ValueType>;
}): State<ValueType> => {
  let baseAtom: StateAtom<ValueType>;

  // Some state belongs in cookies, such as values needed across reloads/domains.
  if (isDefined(useCookieStorage)) {
    baseAtom = atomWithStorage<ValueType>(useCookieStorage.cookieKey, defaultValue, storage, { getOnInit: true }) as StateAtom<ValueType>;
  // Some state belongs only in the current browser tab session.
  } else if (useSessionStorage) {
    baseAtom = atomWithStorage<ValueType>(key, defaultValue, storage, { getOnInit: true }) as StateAtom<ValueType>;
  // Some state can survive browser restart.
  } else if (useLocalStorage) {
    baseAtom = atomWithStorage<ValueType>(key, defaultValue, undefined, localStorageOptions ?? undefined) as StateAtom<ValueType>;
  // Default: memory-only state.
  } else {
    baseAtom = atom(defaultValue);
  }

  baseAtom.debugLabel = key;
  return { type: 'State', key, atom: baseAtom };
};
```

**The "Aha" Moment:**

State should live with the smallest owner that can keep it correct.

**Socratic Checkpoint:**

1. Why is `currentUserState` global?
2. Why should a single input's typed text not become global state?
3. What state does Apollo own that Jotai should not duplicate?
4. What state does the URL own?
5. What bug appears when two owners remember the same fact differently?

Self-grade rubric: strong answers cite `currentUserState.ts:18-21`, `ApolloFactory` client construction at `apollo.factory.ts:359-365`, and route creation at `useCreateAppRouter.tsx:109-260`.

**Connects To:**

- Mission 10, because hooks often encode state ownership.
- Mission 11, because side effects synchronize state with the outside system.

## Mission 10: The Custom Hook Ecosystem

**Tier:** Mid-Level  
**Time Estimate:** 60 minutes  
**Goal:** You will understand how custom hooks hide repeated workflows and where that abstraction helps or hurts.

**The Concept:**

Custom hooks are reusable CRM procedures. Instead of every screen knowing how to create, update, delete, and destroy view sorts, one hook packages the API choreography, error handling, and typed result shape.

**Design Intent (Before You Read the Code):**

1. Files involved: hook inventory under `packages/twenty-front/src/modules/**/hooks`, complex specimen `packages/twenty-front/src/modules/views/hooks/internal/usePerformViewSortAPIPersist.ts:1-223`.
2. The hook owns view-sort persistence operations.
3. Pattern: Apollo `useMutation`, `useCallback`, generated GraphQL variable types, shared metadata error handling.
4. Poor implementation would duplicate mutation/error logic across components and create inconsistent failed-save behavior.

**Find It In The Code:**

Hook groups:

- Auth hooks: `packages/twenty-front/src/modules/auth/hooks/useAuth.ts:1-120`, `packages/twenty-front/src/modules/auth/sign-in-up/hooks`.
- Data/API hooks: `packages/twenty-front/src/modules/views/hooks/internal/usePerformViewSortAPIPersist.ts:1-223`, `packages/twenty-front/src/modules/workspace-invitation/hooks`.
- UI behavior hooks: `packages/twenty-front/src/modules/keyboard-shortcut-menu/hooks/useKeyboardShortcutMenu.ts:1-52`, `packages/twenty-front/src/hooks/useCopyToClipboard.tsx`.
- Metadata hooks: `packages/twenty-front/src/modules/metadata-store/hooks`, `packages/twenty-front/src/modules/object-metadata/hooks`.
- Workflow hooks: `packages/twenty-front/src/modules/workflow/**/hooks`.

```ts
// packages/twenty-front/src/modules/views/hooks/internal/usePerformViewSortAPIPersist.ts:21-30
export const usePerformViewSortAPIPersist = () => {
  // Each generated document becomes a mutation function.
  const [createViewSortMutation] = useMutation(CreateViewSortDocument);
  const [updateViewSortMutation] = useMutation(UpdateViewSortDocument);
  const [deleteViewSortMutation] = useMutation(DeleteViewSortDocument);
  const [destroyViewSortMutation] = useMutation(DestroyViewSortDocument);

  // Error handling is shared so metadata errors feel consistent.
  const { handleMetadataError } = useMetadataErrorHandler();
  const { enqueueErrorSnackBar } = useSnackBar();
```

```ts
// packages/twenty-front/src/modules/views/hooks/internal/usePerformViewSortAPIPersist.ts:30-74
const performViewSortAPICreate = useCallback(
  async (
    // Generated variable types make the caller send the right mutation input shape.
    createViewSortInputs: CreateViewSortMutationVariables[],
  ): Promise<MetadataRequestResult<Awaited<ReturnType<typeof createViewSortMutation>>[]>> => {
    // Empty work is a successful no-op, not an error.
    if (createViewSortInputs.length === 0) {
      return { status: 'successful', response: [] };
    }

    try {
      // Multiple sort changes are persisted concurrently.
      const results = await Promise.all(
        createViewSortInputs.map((variables) =>
          createViewSortMutation({ variables }),
        ),
      );
      return { status: 'successful', response: results };
    } catch (error) {
      // GraphQL errors get metadata-aware handling; unknown errors get a generic snackbar.
      if (CombinedGraphQLErrors.is(error)) {
        handleMetadataError(error, {
          primaryMetadataName: 'viewSort',
          operationType: CrudOperationType.CREATE,
        });
      } else {
        enqueueErrorSnackBar({ message: t`An error occurred.` });
      }
      return { status: 'failed', error };
    }
  },
  // Dependencies list every outside value the callback closes over.
  [createViewSortMutation, handleMetadataError, enqueueErrorSnackBar],
);
```

Without the hook, every component that saves view sorts would manually call four mutations, check empty arrays, catch GraphQL errors, call metadata error handlers, enqueue snackbars, and return consistent result objects.

**The "Aha" Moment:**

A good hook is not just reused code; it gives a workflow one owner.

**Socratic Checkpoint:**

1. What complexity does `usePerformViewSortAPIPersist` hide?
2. Why are generated mutation variable types important?
3. Why is empty input treated as success?
4. What dependency would be dangerous to omit?
5. What would make this hook too broad?

Self-grade rubric: strong answers cite lines `21-30`, `30-74`, and note the repeated CRUD pattern through `76-223`.

**Connects To:**

- Mission 11, because hook callbacks and effects both rely on dependencies.
- Mission 14, because this hook can start an end-to-end metadata trace.

## Mission 11: Side Effects Are Promises to the System

**Tier:** Mid-Level  
**Time Estimate:** 55 minutes  
**Goal:** You will understand `useEffect` dependencies through real workspace redirect effects.

**The Concept:**

A side effect is a promise to synchronize React with something outside render: URL, browser storage, network, timers, subscriptions, or another tab. In Twenty, workspace effects make sure a user standing at the wrong CRM domain is moved to the right workspace door.

**Design Intent (Before You Read the Code):**

1. Files involved: `packages/twenty-front/src/modules/workspace/components/WorkspaceProviderEffect.tsx:18-90`, plus important effect files such as `AppRouterProviders.tsx:46-50`, `PromiseRejectionEffect.tsx:1-60`, `PageChangeEffect.tsx:1-80`, `SignOutOnOtherTabSignOutEffect.tsx:1-80`.
2. `WorkspaceProviderEffect` owns domain/workspace redirect synchronization.
3. Pattern: read state/hooks, compute helper with `useCallback`, run guarded effects with explicit dependencies.
4. Poor implementation causes redirect loops, stale domains, lost query params, or wrong workspace after login.

**Find It In The Code:**

```tsx
// packages/twenty-front/src/modules/workspace/components/WorkspaceProviderEffect.tsx:36-42
// This callback depends on currentLocationHostname; if hostname changes, the comparison must change too.
const isWorkspaceHostnameMatchCurrentLocationHostname = useCallback(
  (workspaceUrls: WorkspaceUrls) => {
    const { hostname } = new URL(getWorkspaceUrl(workspaceUrls));
    return hostname === currentLocationHostname;
  },
  [currentLocationHostname],
);
```

```tsx
// packages/twenty-front/src/modules/workspace/components/WorkspaceProviderEffect.tsx:44-64
useEffect(() => {
  // Only redirect when multi-workspace is enabled, data loaded, and current host is wrong.
  if (
    isMultiWorkspaceEnabled &&
    isDefined(getPublicWorkspaceData) &&
    !isWorkspaceHostnameMatchCurrentLocationHostname(
      getPublicWorkspaceData.workspaceUrls,
    )
  ) {
    // Preserve path and query params while moving to the correct workspace domain.
    redirectToWorkspaceDomain(
      getWorkspaceUrl(getPublicWorkspaceData.workspaceUrls),
      window.location.pathname,
      getCurrentSearchParams(),
    );
  }
}, [
  // Each dependency is a fact the effect reads from outside its body.
  isMultiWorkspaceEnabled,
  redirectToWorkspaceDomain,
  getPublicWorkspaceData,
  currentLocationHostname,
  isWorkspaceHostnameMatchCurrentLocationHostname,
]);
```

```tsx
// packages/twenty-front/src/modules/workspace/components/WorkspaceProviderEffect.tsx:66-87
useEffect(() => {
  // If user is on default domain and has a remembered workspace, redirect there.
  if (
    isMultiWorkspaceEnabled &&
    isDefaultDomain &&
    isDefined(lastAuthenticatedWorkspaceDomain) &&
    'workspaceUrl' in lastAuthenticatedWorkspaceDomain &&
    isDefined(lastAuthenticatedWorkspaceDomain?.workspaceUrl)
  ) {
    // Initialize URL-synced state before redirecting.
    initializeQueryParamState();
    redirectToWorkspaceDomain(
      lastAuthenticatedWorkspaceDomain.workspaceUrl,
      window.location.pathname,
      getCurrentSearchParams(),
    );
  }
}, [
  isMultiWorkspaceEnabled,
  isDefaultDomain,
  lastAuthenticatedWorkspaceDomain,
  redirectToWorkspaceDomain,
  initializeQueryParamState,
]);
```

Five important effects to inspect:

1. `WorkspaceProviderEffect.tsx:44-64` redirects when public workspace data host does not match.
2. `WorkspaceProviderEffect.tsx:66-87` redirects from default domain to last workspace.
3. `packages/twenty-front/src/modules/app/components/AppRouterProviders.tsx:46-50` mounts config/metadata/workspace loading effects.
4. `packages/twenty-front/src/modules/error-handler/components/PromiseRejectionEffect.tsx:1-60` catches unhandled promise rejections.
5. `packages/twenty-front/src/modules/auth/effect-components/SignOutOnOtherTabSignOutEffect.tsx:1-80` coordinates cross-tab sign out.

**The "Aha" Moment:**

Dependency arrays are not performance hints; they are the truth table for when React must resynchronize with the outside world.

**Socratic Checkpoint:**

1. Why does the first effect depend on `getPublicWorkspaceData`?
2. What redirect bug could happen if `currentLocationHostname` were stale?
3. Why initialize query-param state before redirecting?
4. What kind of cleanup would an event-listener effect need?
5. Why is an effect with missing dependencies often worse than an effect that runs too often?

Self-grade rubric: strong answers describe stale closure risk, redirect loops, and external synchronization.

**Connects To:**

- Mission 13, because middleware is the backend version of pre-handler synchronization.
- Mission 21, because effects can cause performance issues.

## Mission 12: The Full API Contract

**Tier:** Mid-Level  
**Time Estimate:** 90 minutes  
**Goal:** You will map backend API families and learn how to review their consistency.

**The Concept:**

An API contract is the menu of services the CRM exposes. Twenty has several menus: GraphQL for app operations, metadata GraphQL for workspace definitions, REST for dynamic object records, metadata REST for metadata entities, MCP for tool clients, and specialized product endpoints.

**Design Intent (Before You Read the Code):**

1. Files involved: `packages/twenty-server/src/app.module.ts:52-70`, `packages/twenty-server/src/engine/api/rest/core/controllers/rest-api-core.controller.ts:24-163`, `packages/twenty-server/src/engine/metadata-modules/view-sort/controllers/view-sort.controller.ts:38-126`, `packages/twenty-server/src/engine/metadata-modules/webhook/controllers/webhook.controller.ts:28-94`, `packages/twenty-server/src/modules/dashboard/controllers/dashboard.controller.ts:11-24`.
2. Controllers/resolvers own external route shapes; services own behavior.
3. Pattern: controller decorator, guards, filters, typed DTO/input, service method.
4. Poor API design creates inconsistent auth, unclear request shapes, drifting frontend calls, and hard-to-test failures.

**Find It In The Code:**

```ts
// API family registration: packages/twenty-server/src/app.module.ts:52-70
GraphQLModule.forRootAsync<YogaDriverConfig>({...}); // /graphql-style APIs
CoreGraphQLApiModule;                                // core GraphQL
MetadataGraphQLApiModule;                            // metadata GraphQL
AdminPanelGraphQLApiModule;                          // admin GraphQL
RestApiModule;                                       // REST controllers
McpModule;                                           // MCP endpoint
```

Representative route table:

| Method | Path | File lines | Request shape | Response shape | Auth |
|---|---|---|---|---|---|
| POST | `/rest/dashboards/:id/duplicate` | `dashboard.controller.ts:19-24` | `id: string` param | `DuplicatedDashboardDTO` | `JwtAuthGuard`, `WorkspaceAuthGuard`, `NoPermissionGuard` at lines 11-13 |
| GET | `/rest/*path` | `rest-api-core.controller.ts:83-94` | `AuthenticatedRequest` | dynamic object result | `JwtAuthGuard`, `WorkspaceAuthGuard`, `CustomPermissionGuard` at lines 24-26 |
| POST | `/rest/*path` | `rest-api-core.controller.ts:57-68` | `AuthenticatedRequest` body/path | dynamic create result | same |
| PATCH | `/rest/*path` | `rest-api-core.controller.ts:135-146` | `AuthenticatedRequest` | dynamic update result | same |
| GET | `/rest/metadata/viewSorts` | `view-sort.controller.ts:48-59` | optional `viewId` query | `ViewSortDTO[]` | `WorkspaceAuthGuard`, `NoPermissionGuard` |
| POST | `/rest/metadata/viewSorts` | `view-sort.controller.ts:87-97` | `CreateViewSortInput` | `ViewSortDTO` | `CreateViewSortPermissionGuard` |
| PATCH | `/rest/metadata/viewSorts/:id` | `view-sort.controller.ts:99-110` | `{ direction?: ViewSortDirection }` | `ViewSortDTO` | `UpdateViewSortPermissionGuard` |
| GET | `/rest/webhooks` and `/rest/metadata/webhooks` | `webhook.controller.ts:42-47` | none | `WebhookDTO[]` | JWT, workspace, settings permission |
| PATCH | `/rest/webhooks/:id` | `webhook.controller.ts:65-83` | target URL/operations/description/secret | `WebhookDTO` | JWT, workspace, settings permission |

Frontend call matching examples:

- Apollo client/link construction: `packages/twenty-front/src/modules/apollo/services/apollo.factory.ts:98-109`.
- View sort mutations: `packages/twenty-front/src/modules/views/hooks/internal/usePerformViewSortAPIPersist.ts:21-25`.
- Current user GraphQL query: `packages/twenty-front/src/modules/users/hooks/useLoadCurrentUser.ts:48-51`.

Ratings:

- Consistency: 3/5. Metadata controllers are consistent; dynamic REST routes are flexible but less self-documenting.
- Type safety: 3/5. GraphQL/generated types are strong; wildcard `AuthenticatedRequest` routes rely on runtime parsing.
- Error handling: 4/5. Filters are consistently attached, such as `DashboardRestApiExceptionFilter` at `dashboard.controller.ts:13` and view-sort filters at `view-sort.controller.ts:40-44`.

**The "Aha" Moment:**

In a metadata-driven app, the API contract is partly declared in controllers and partly generated from workspace metadata.

**Socratic Checkpoint:**

1. Why are wildcard REST routes harder to review than dashboard duplication?
2. Which endpoint family does Apollo use for metadata mutations?
3. What does a guard prove?
4. Why do filters matter to the contract?
5. Where would you add a new typed metadata endpoint?

Self-grade rubric: strong answers distinguish GraphQL, metadata GraphQL, dynamic REST, metadata REST, and specialized product endpoints.

**Connects To:**

- Mission 13, because middleware prepares API requests.
- Mission 14, because a feature trace crosses the API contract.

## Mission 13: The Middleware Chain

**Tier:** Mid-Level  
**Time Estimate:** 60 minutes  
**Goal:** You will trace an authenticated request before it reaches a handler.

**The Concept:**

Middleware is the CRM security/front-desk process before a request reaches a department. It reads badges, attaches workspace context, rejects invalid methods, and enriches requests so controllers can stay focused.

**Design Intent (Before You Read the Code):**

1. Files involved: `packages/twenty-server/src/main.ts:56-88`, `packages/twenty-server/src/app.module.ts:114-146`.
2. `main.ts` installs global Express/Nest middleware. `AppModule.configure` installs route-specific middleware.
3. Pattern: global middleware first, route middleware next, guards at controller/method, then service.
4. Poor ordering can leave `workspaceId` missing, skip auth context, break uploads, or reject valid clients.

**Find It In The Code:**

```ts
// packages/twenty-server/src/main.ts:56-88
// Session middleware runs globally.
app.use(session(getSessionStorageOptions(twentyConfigService)));

// Validation can use Nest dependency injection.
useContainer(app.select(AppModule), { fallbackOnErrors: true });

// Global exception handling catches unhandled failures.
app.useGlobalFilters(new UnhandledExceptionFilter());

// Body parsers define what request payloads Nest can read.
app.useBodyParser('json', { limit: settings.storage.maxFileSize });
app.useBodyParser('urlencoded', { limit: settings.storage.maxFileSize, extended: true });
app.useBodyParser('text', { type: 'text/plain', limit: '1024kb' });

// Upload middleware runs on GraphQL endpoints that accept files.
app.use('/graphql', graphqlUploadExpress({ maxFieldSize: bytes(settings.storage.maxFileSize)!, maxFiles: 10 }));
app.use('/metadata', graphqlUploadExpress({ maxFieldSize: bytes(settings.storage.maxFileSize)!, maxFiles: 10 }));
```

```ts
// packages/twenty-server/src/app.module.ts:114-146
configure(consumer: MiddlewareConsumer) {
  // GraphQL requests: hydrate token, then attach workspace auth context.
  consumer
    .apply(GraphQLHydrateRequestFromTokenMiddleware, WorkspaceAuthContextMiddleware)
    .forRoutes({ path: 'graphql', method: RequestMethod.ALL });

  // Metadata and admin-panel requests get the same auth/workspace setup.
  consumer
    .apply(GraphQLHydrateRequestFromTokenMiddleware, WorkspaceAuthContextMiddleware)
    .forRoutes({ path: 'metadata', method: RequestMethod.ALL });

  // MCP has method guard middleware.
  consumer
    .apply(McpMethodGuardMiddleware)
    .forRoutes({ path: 'mcp', method: RequestMethod.ALL });

  // REST object routes get REST parsing before workspace context.
  for (const method of MIGRATED_REST_METHODS) {
    consumer
      .apply(RestCoreMiddleware, WorkspaceAuthContextMiddleware)
      .forRoutes({ path: 'rest/*path', method });
  }
}
```

Authenticated REST trace:

```text
HTTP request /rest/...
  -> global session/body parser
  -> RestCoreMiddleware
  -> WorkspaceAuthContextMiddleware
  -> controller guards such as JwtAuthGuard / WorkspaceAuthGuard
  -> controller method
  -> service
```

**The "Aha" Moment:**

Middleware makes controller assumptions true.

**Socratic Checkpoint:**

1. Why does REST use `RestCoreMiddleware` before `WorkspaceAuthContextMiddleware`?
2. What breaks if upload middleware is removed from `/metadata`?
3. What is the difference between middleware and guards?
4. Why does middleware order matter?
5. Where is request context attached for GraphQL?

Self-grade rubric: strong answers cite `main.ts:56-88` and `app.module.ts:114-146`.

**Connects To:**

- Mission 14, because the feature trace follows one request through middleware.
- Mission 22, because security review starts at middleware and guards.

## Mission 14: End-to-End Feature Trace

**Tier:** Mid-Level  
**Time Estimate:** 90 minutes  
**Goal:** You will trace one realistic feature from frontend trigger to backend persistence and response.

**The Concept:**

End-to-end tracing is following a CRM paperwork request from the user's click, through the front office, through the API desk, into the records department, and back to the screen.

**Design Intent (Before You Read the Code):**

1. Files involved: `usePerformViewSortAPIPersist.ts:21-223`, `apollo.factory.ts:111-140`, `view-sort.controller.ts:87-110`, `view-sort.service.ts:34-170`, `app.module.ts:114-146`.
2. Frontend hook owns mutation trigger. Apollo owns HTTP formation. Controller owns API boundary. Service owns metadata migration.
3. Pattern: generated mutation -> Apollo link -> Nest middleware/guard -> controller -> service -> migration/cache -> DTO.
4. Poor implementation can create unauthorized changes, stale UI, failed migrations, or inconsistent sort state.

**Find It In The Code:**

Trace: create or update a view sort.

1. React event handler: find the caller of `performViewSortAPICreate` via `rg "performViewSortAPI" packages/twenty-front/src/modules/views -n`.
2. Data trigger: hook exposes functions at `packages/twenty-front/src/modules/views/hooks/internal/usePerformViewSortAPIPersist.ts:217-222`.
3. HTTP request: Apollo adds headers at `packages/twenty-front/src/modules/apollo/services/apollo.factory.ts:111-140`.
4. Middleware: backend applies GraphQL/REST middleware at `packages/twenty-server/src/app.module.ts:114-146`.
5. Route handler: `ViewSortController.create` at `packages/twenty-server/src/engine/metadata-modules/view-sort/controllers/view-sort.controller.ts:87-97`.
6. Service: `ViewSortService.createOne` at `packages/twenty-server/src/engine/metadata-modules/view-sort/services/view-sort.service.ts:34-101`.
7. Persistence/migration: `validateBuildAndRunWorkspaceMigration` at `view-sort.service.ts:63-78`.
8. Response formation: `fromFlatViewSortToViewSortDto` at `view-sort.service.ts:95-100`.

```ts
// packages/twenty-front/src/modules/views/hooks/internal/usePerformViewSortAPIPersist.ts:45-56
// The hook sends one mutation per pending view-sort create input.
const results = await Promise.all(
  createViewSortInputs.map((variables) =>
    createViewSortMutation({
      variables,
    }),
  ),
);
return {
  status: 'successful',
  response: results,
};
```

```ts
// packages/twenty-server/src/engine/metadata-modules/view-sort/controllers/view-sort.controller.ts:87-97
@Post()
@UseGuards(CreateViewSortPermissionGuard)
async create(
  @Body() input: CreateViewSortInput,
  @AuthWorkspace() { id: workspaceId }: WorkspaceEntity,
): Promise<ViewSortDTO> {
  // The controller passes typed input and workspace ID into the service.
  return await this.viewSortService.createOne({
    createViewSortInput: input,
    workspaceId,
  });
}
```

```ts
// packages/twenty-server/src/engine/metadata-modules/view-sort/services/view-sort.service.ts:56-100
// Convert API input into the flat metadata entity the migration system understands.
const flatViewSortToCreate = fromCreateViewSortInputToFlatViewSortToCreate({
  createViewSortInput,
  flatApplication: workspaceCustomFlatApplication,
  flatFieldMetadataMaps,
  flatViewMaps,
});

// Validate/build/run the metadata migration instead of directly inserting a row.
const buildAndRunResult =
  await this.workspaceMigrationValidateBuildAndRunService.validateBuildAndRunWorkspaceMigration({
    allFlatEntityOperationByMetadataName: {
      viewSort: {
        flatEntityToCreate: [flatViewSortToCreate],
        flatEntityToDelete: [],
        flatEntityToUpdate: [],
      },
    },
    workspaceId,
    isSystemBuild: false,
    applicationUniversalIdentifier: workspaceCustomFlatApplication.universalIdentifier,
  });

// Re-read metadata maps and return the DTO shape expected by the API.
return fromFlatViewSortToViewSortDto(
  findFlatEntityByIdInFlatEntityMapsOrThrow({
    flatEntityId: flatViewSortToCreate.id,
    flatEntityMaps: recomputedExistingFlatViewSortMaps,
  }),
);
```

**The "Aha" Moment:**

In this codebase, metadata writes are not simple database writes; they are validated workspace migrations.

**Socratic Checkpoint:**

1. What starts the view-sort API operation?
2. Where is the request authorized?
3. Why does the service convert input into a flat entity?
4. Why recompute maps after migration?
5. What would break if the service inserted directly into the repository?

Self-grade rubric: strong answers include frontend mutation, Apollo headers, middleware/guards, service migration, and DTO response.

**Connects To:**

- Mission 15, because this trace is a realistic PR review area.
- Mission 18, because metadata-driven migrations are an architecture decision.

## Mission 15: Read the Diff Like an Engineer

**Tier:** Mid-Level  
**Time Estimate:** 45 minutes  
**Goal:** You will practice reviewing a realistic change by risk, contract, and blast radius.

**The Concept:**

Reading a diff is not reading lines top to bottom. It is reconstructing intent, affected contracts, assumptions, and failure modes.

**Design Intent (Before You Read the Code):**

1. Files involved: hypothetical change to `view-sort.controller.ts:99-110`, `view-sort.service.ts:103-170`, frontend hook `usePerformViewSortAPIPersist.ts:76-121`, generated metadata types.
2. The change adds an optional `nullsFirst` setting to view sort update.
3. Pattern: DTO/schema -> frontend generated type -> hook variables -> service conversion -> migration validation.
4. Poor implementation lets UI send a field backend ignores, or backend accepts field frontend cannot type.

**Find It In The Code:**

Hypothetical diff:

```diff
// packages/twenty-server/src/engine/metadata-modules/view-sort/controllers/view-sort.controller.ts:99-110
- @Body() input: { direction?: ViewSortDirection },
+ @Body() input: { direction?: ViewSortDirection; nullsFirst?: boolean },
```

Review order:

1. Backend input/controller: `view-sort.controller.ts:99-110`.
2. Service conversion: `view-sort.service.ts:125-130`.
3. Flat view sort utilities under `packages/twenty-server/src/engine/metadata-modules/flat-view-sort/utils`.
4. GraphQL schema/resolver if metadata GraphQL also exposes update.
5. Generated frontend metadata types: `packages/twenty-front/src/generated-metadata/graphql.ts`.
6. Frontend hook: `usePerformViewSortAPIPersist.ts:76-121`.
7. Tests for create/update/delete view sort.

**The "Aha" Moment:**

A good reviewer follows the contract outward from the changed boundary.

**Socratic Checkpoint:**

1. What is the contract change?
2. Which files must agree on the new field?
3. What runtime bug happens if frontend sends `nullsFirst` but backend ignores it?
4. What test would catch stale generated types?
5. What is safe to skim?

Self-grade rubric: strong answers name controller, DTO/schema, flat utils, service, generated types, frontend hook, and tests.

**Connects To:**

- Mission 17, because TypeScript contract changes can be non-obvious.
- Mission 24, because good commits tell the same contract story.

## Mission 16: Composition Over Inheritance

**Tier:** Mid-Level  
**Time Estimate:** 50 minutes  
**Goal:** You will understand React composition through the app provider tree.

**The Concept:**

Composition means building the CRM workspace shell by nesting capabilities instead of creating one massive base class. `AppRouterProviders` composes Apollo, theme, metadata, auth, SSE, command menu, dialogs, and page outlet like layered services around each page.

**Design Intent (Before You Read the Code):**

1. File involved: `packages/twenty-front/src/modules/app/components/AppRouterProviders.tsx:39-96`.
2. It owns the runtime environment for all routes.
3. Pattern: provider nesting, effect components, context providers, outlet.
4. Poor implementation creates provider order bugs, missing context, or broad re-render pressure.

**Find It In The Code:**

```tsx
// packages/twenty-front/src/modules/app/components/AppRouterProviders.tsx:43-96
return (
  <ApolloProvider>
    <BaseThemeProvider>
      // Effects load config, metadata, user/workspace state before or during route rendering.
      <ClientConfigProviderEffect />
      <UserMetadataProviderInitialEffect />
      <MinimalMetadataLoadEffect />
      <WorkspaceProviderEffect />
      <ClientConfigProvider>
        <CaptchaProvider>
          <MinimalMetadataGater>
            <AuthProvider>
              <ApolloCoreProvider>
                <ApolloAdminProvider>
                  <SSEProvider>
                    <PreComputedChipGeneratorsProvider>
                      <SnackBarProvider>
                        <AgentChatProvider>
                          <DialogManager>
                            <StrictMode>
                              // Outlet is where the matched route page appears.
                              <Outlet />
                              <GlobalFilePreviewModal />
                              <CommandMenuConfirmationModalManager />
                              <CommandRunner />
                            </StrictMode>
                          </DialogManager>
                        </AgentChatProvider>
                      </SnackBarProvider>
                      <MainContextStoreProvider />
                      <SupportChatEffect />
                      <PageChangeEffect />
                      <SignOutOnOtherTabSignOutEffect />
                    </PreComputedChipGeneratorsProvider>
                  </SSEProvider>
                </ApolloAdminProvider>
              </ApolloCoreProvider>
            </AuthProvider>
          </MinimalMetadataGater>
        </CaptchaProvider>
      </ClientConfigProvider>
    </BaseThemeProvider>
  </ApolloProvider>
);
```

Alternative would be a huge `AppRuntime` class/object manually passing services into every page. React composition lets each provider own one concern.

**The "Aha" Moment:**

Composition scales by making each layer responsible for one capability and letting children consume only what they need.

**Socratic Checkpoint:**

1. Why is `<Outlet />` deep in the tree?
2. What provider must exist before components use Apollo hooks?
3. Why are some items "effect components" with no UI?
4. What makes this composition powerful?
5. What makes this composition risky?

Self-grade rubric: strong answers discuss capability layering, provider order, and context dependencies.

**Connects To:**

- Mission 18, because this is a major architecture decision.
- Mission 21, because provider composition can affect rendering performance.

## Mission 17: TypeScript's Hidden Work

**Tier:** Mid-Level  
**Time Estimate:** 50 minutes  
**Goal:** You will spot non-obvious TypeScript patterns that prevent real bugs.

**The Concept:**

Advanced TypeScript is the CRM's rulebook for complicated forms. It does not just label strings; it shapes allowed operations, narrows unknown errors, and connects generated API contracts to real functions.

**Design Intent (Before You Read the Code):**

1. Files involved: `createAtomState.ts:8-79`, `usePerformViewSortAPIPersist.ts:30-37`, `apollo.factory.ts:49-66`, `currentUserState.ts:4-21`.
2. These files use generics, utility types, return-type inference, union/null types, and type guards.
3. Pattern: type the boundary where mistakes are expensive.
4. Poor implementation makes runtime assumptions invisible.

**Find It In The Code:**

Examples:

```ts
// createAtomState.ts:27-41
// Generic ValueType means the atom's value and default value must agree.
export const createAtomState = <ValueType>({
  key,
  defaultValue,
}: {
  key: string;
  defaultValue: ValueType;
}): State<ValueType> => { ... };
```

```ts
// currentUserState.ts:4-18
// Pick prevents global state from silently becoming the entire generated User object.
export type CurrentUser = Pick<User, 'id' | 'email' | 'canImpersonate' | 'hasPassword'>;
export const currentUserState = createAtomState<CurrentUser | null>({...});
```

```ts
// usePerformViewSortAPIPersist.ts:30-37
// Awaited<ReturnType<...>> ties the hook's result type to the actual mutation function.
): Promise<
  MetadataRequestResult<
    Awaited<ReturnType<typeof createViewSortMutation>>[]
  >
> => { ... }
```

```ts
// apollo.factory.ts:49-66
// Options makes ApolloFactory construction explicit and reviewable.
export interface Options {
  uri: string;
  cache: ApolloClient.Options['cache'];
  onTokenPairChange?: (tokenPair: AuthTokenPair) => void;
  currentWorkspaceMember: CurrentWorkspaceMember | null;
  currentWorkspace: CurrentWorkspace | null;
}
```

**The "Aha" Moment:**

The best TypeScript lives at boundaries: state, API, errors, and reusable abstractions.

**Socratic Checkpoint:**

1. What bug does `ValueType` prevent in `createAtomState`?
2. Why is `CurrentUser | null` better than `CurrentUser`?
3. What does `Awaited<ReturnType<...>>` keep synchronized?
4. Why does `Options` make `ApolloFactory` safer?
5. Where does TypeScript still fall short in this app?

Self-grade rubric: strong answers mention JSON/Upload `any` in `packages/twenty-front/src/generated-metadata/graphql.ts:14-19` as a place where runtime validation still matters.

**Connects To:**

- Mission 19, because types that lie can become bugs.
- Mission 22, because unsafe input assumptions are security risks.

