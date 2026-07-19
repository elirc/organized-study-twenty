# Tier 3 - Senior Missions

## Mission 18: Reverse-Engineer the Architecture Decisions

**Tier:** Senior  
**Time Estimate:** 75 minutes  
**Goal:** You will reconstruct why the codebase is shaped this way and what each decision costs.

**The Concept:**

Senior engineering is reading code as a series of tradeoffs. Twenty's architecture says the team prioritized extensible workspace metadata, centralized runtime capabilities, and consistent API middleware over local simplicity.

**Design Intent (Before You Read the Code):**

1. Files involved: `packages/twenty-front/src/modules/app/components/AppRouterProviders.tsx:39-96`, `packages/twenty-front/src/modules/apollo/services/apollo.factory.ts:98-365`, `packages/twenty-server/src/app.module.ts:49-146`, `packages/twenty-server/src/engine/api/rest/core/controllers/rest-api-core.controller.ts:24-163`, `packages/twenty-server/src/engine/metadata-modules/view-sort/services/view-sort.service.ts:34-101`.
2. These files reveal decisions: provider shell, centralized Apollo, Nest modular engine, dynamic REST, migration-backed metadata.
3. Pattern: centralize cross-cutting concerns; keep feature code thinner.
4. Poor decision-making here would make the app faster to start but harder to evolve safely.

**Find It In The Code:**

Five decisions:

1. Provider-heavy frontend shell: `AppRouterProviders.tsx:43-96`.
   - Alternative: each page imports its own providers.
   - Prioritized: consistency and capability availability.
   - Cost: boot complexity and provider-order risk.
2. Centralized Apollo transport: `apollo.factory.ts:98-365`.
   - Alternative: per-feature fetch clients.
   - Prioritized: consistent auth, retry, upload, REST, error handling.
   - Cost: request debugging requires link-chain knowledge.
3. Nest root module composition: `app.module.ts:49-146`.
   - Alternative: smaller standalone Express routers.
   - Prioritized: DI, modules, guards, middleware, testable services.
   - Cost: more framework concepts to learn.
4. Dynamic REST object API: `rest-api-core.controller.ts:24-163`.
   - Alternative: one controller per CRM object.
   - Prioritized: custom objects and metadata-driven flexibility.
   - Cost: weaker local route readability.
5. Metadata migration services: `view-sort.service.ts:63-78`.
   - Alternative: direct repository writes.
   - Prioritized: workspace schema consistency.
   - Cost: more indirection for simple-looking changes.

```ts
// packages/twenty-server/src/engine/metadata-modules/view-sort/services/view-sort.service.ts:63-78
// This is an architecture decision: metadata changes are validated migrations.
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
```

**The "Aha" Moment:**

Architecture is what the codebase makes easy and what it makes expensive.

**Socratic Checkpoint:**

1. Which decision most improves scalability?
2. Which decision most increases onboarding cost?
3. Why is dynamic REST reasonable for a CRM?
4. Which decision would you revisit first?
5. What decision protects workspace metadata consistency?

Self-grade rubric: strong answers discuss tradeoffs, not just labels.

**Connects To:**

- Mission 19, because tradeoffs create latent bug patterns.
- Mission 25, because ownership docs should preserve decision reasoning.

## Mission 19: Find the Bugs Before They Happen

**Tier:** Senior  
**Time Estimate:** 70 minutes  
**Goal:** You will identify not-yet-bugs that could fail under pressure.

**The Concept:**

Senior debugging starts before the bug report. You look for code that is correct only if timing, permissions, generated types, or data shape assumptions remain favorable.

**Design Intent (Before You Read the Code):**

1. Files involved: `apollo.factory.ts:41-44`, `apollo.factory.ts:162-210`, `generated-metadata/graphql.ts:14-19`, `WorkspaceProviderEffect.tsx:44-87`, `webhook.controller.ts:65-83`.
2. These areas involve concurrency, unsafe scalar types, redirects, and manual input shapes.
3. Pattern: centralize risk but test it heavily.
4. Poor implementation creates race conditions, redirects loops, runtime JSON crashes, or API drift.

**Find It In The Code:**

Potential issue 1: token renewal race.

```ts
// packages/twenty-front/src/modules/apollo/services/apollo.factory.ts:41-44
// Shared promise dedupes concurrent renewal attempts.
let renewalPromise: Promise<boolean> | null = null;
```

Scenario: if future edits make this instance-local instead of module-global, concurrent `/graphql` and `/metadata` failures could trigger duplicate renewals and inconsistent auth state.

Potential issue 2: generated `any` scalars.

```ts
// packages/twenty-front/src/generated-metadata/graphql.ts:14-19
ConnectionCursor: any;
JSON: any;
JSONObject: any;
Upload: any;
```

Scenario: a malformed JSON field passes TypeScript and crashes when a component assumes object shape.

Potential issue 3: workspace redirect loop.

```tsx
// packages/twenty-front/src/modules/workspace/components/WorkspaceProviderEffect.tsx:44-64
useEffect(() => {
  if (isMultiWorkspaceEnabled && isDefined(getPublicWorkspaceData) && !isWorkspaceHostnameMatchCurrentLocationHostname(...)) {
    redirectToWorkspaceDomain(...);
  }
}, [isMultiWorkspaceEnabled, redirectToWorkspaceDomain, getPublicWorkspaceData, currentLocationHostname, isWorkspaceHostnameMatchCurrentLocationHostname]);
```

Scenario: incorrect hostname parsing or stale public workspace data can redirect repeatedly.

Potential issue 4: REST body drift.

```ts
// packages/twenty-server/src/engine/metadata-modules/webhook/controllers/webhook.controller.ts:65-83
@Body()
updateWebhookDto: {
  targetUrl?: string;
  operations?: string[];
  description?: string;
  secret?: string;
},
```

Scenario: GraphQL `UpdateWebhookInput` evolves, REST manual body shape does not.

**Corrected version example:**

```ts
// Prefer sharing the DTO/input shape where possible so REST and GraphQL do not drift.
type UpdateWebhookRestBody = UpdateWebhookInput['update'];
```

**The "Aha" Moment:**

Latent bugs hide where runtime assumptions are stronger than the code's proof.

**Socratic Checkpoint:**

1. Why is shared `renewalPromise` important?
2. Where can generated `any` become dangerous?
3. What makes redirect effects prone to loops?
4. Why can manual REST body types drift?
5. Which latent bug would you test first?

Self-grade rubric: strong answers name trigger condition, impact, and a test.

**Connects To:**

- Mission 20, because bug injection trains this intuition.
- Mission 23, because risky latent bugs need tests.

## Mission 20: The Bug Injection Challenge

**Tier:** Senior  
**Time Estimate:** 60 minutes  
**Goal:** You will practice debugging from symptoms before seeing the bug location.

**The Concept:**

Bug injection is controlled failure practice. You do not learn debugging by reading fixes; you learn by forming hypotheses from symptoms and then proving or disproving them in code.

**Design Intent (Before You Read the Code):**

1. Files likely involved: auth/Apollo, workspace redirects, metadata services, guards, generated JSON consumers.
2. The exercise should not actually modify the repo unless you intentionally create a training branch.
3. Pattern: symptom -> hypothesis -> search path -> proof -> fix -> regression test.
4. Poor debugging opens random files without a theory.

**Find It In The Code:**

Training bugs to inject later on a disposable branch:

1. Symptom: two expired-token requests sign the user out even though renewal should succeed.
   - Test scenario: mock concurrent `UNAUTHENTICATED` responses and assert one renewal.
2. Symptom: a non-admin user can reach an admin settings route by direct URL.
   - Test scenario: load settings route with `currentUser.canAccessFullAdminPanel=false`.
3. Symptom: webhook update accepts a bad URL shape and later webhook delivery fails.
   - Test scenario: `PATCH /rest/webhooks/:id` with malformed target URL and assert validation.
4. Symptom: a saved view sort appears until refresh, then disappears.
   - Test scenario: create view sort, reload metadata, assert it persists.
5. Symptom: workspace redirect preserves path but drops query params.
   - Test scenario: visit default domain with query params and assert redirected URL includes them.

Do not reveal bug locations during the exercise. Use this review prompt after investigation:

```text
I investigated the bugs. Here are my notes: [YOUR NOTES]

Evaluate my approach:
1. Did I look in the right files?
2. Did I form a reasonable hypothesis before opening the code?
3. What did I miss or overlook?
4. What would a senior engineer have checked first?
5. Walk me through the correct debugging path for each bug.
```

**The "Aha" Moment:**

The first debugging skill is choosing where to look before touching the keyboard.

**Socratic Checkpoint:**

1. Which symptom points to Apollo?
2. Which symptom points to router state?
3. Which symptom points to backend validation?
4. Which symptom points to metadata persistence?
5. Which symptom points to URL/query handling?

Self-grade rubric: strong answers map symptom to likely subsystem and first file.

**Connects To:**

- Mission 21, because performance debugging also starts with symptoms.
- Mission 23, because every injected bug should end as a regression test.

## Mission 21: Performance X-Ray

**Tier:** Senior  
**Time Estimate:** 75 minutes  
**Goal:** You will identify likely frontend and backend performance risks and propose fixes.

**The Concept:**

Performance review asks where the CRM does more work than the user action justifies: too many renders, too much provider churn, too many network calls, too much metadata recompute, or too much DB work per request.

**Design Intent (Before You Read the Code):**

1. Files involved: `AppRouterProviders.tsx:43-96`, `useApolloFactory.ts:45-119`, `view-sort.service.ts:87-100`, workflow cron query files such as `workflow-cron-trigger-cron.job.ts:154-175`.
2. Frontend risk: broad provider tree and Apollo client recreation. Backend risk: metadata recompute and raw queries.
3. Pattern: centralize expensive work and make it observable.
4. Poor implementation degrades every route or every workspace request.

**Find It In The Code:**

Finding 1: provider stack breadth.

```tsx
// packages/twenty-front/src/modules/app/components/AppRouterProviders.tsx:43-96
// Many providers/effects mount above every route.
// Fix direction: split by concern and profile which providers truly need to wrap all pages.
```

Finding 2: Apollo client creation.

```ts
// packages/twenty-front/src/modules/apollo/hooks/useApolloFactory.ts:45-119
// If dependencies are expanded casually, this useMemo can recreate the Apollo client and clear cache.
const apolloClient = useMemo(() => {
  apolloRef.current = new ApolloFactory({...});
  return apolloRef.current.getClient();
}, [setTokenPair, setCurrentUser, setCurrentWorkspaceMember, setCurrentWorkspace, setReturnToPath, enqueueErrorSnackBar]);
```

Finding 3: metadata recompute after writes.

```ts
// packages/twenty-server/src/engine/metadata-modules/view-sort/services/view-sort.service.ts:87-100
// Recomputing maps after every metadata write may become expensive under batch operations.
const { flatViewSortMaps: recomputedExistingFlatViewSortMaps } =
  await this.flatEntityMapsCacheService.getOrRecomputeManyOrAllFlatEntityMaps({...});
```

Corrective snippet:

```ts
// Add timing around expensive recomputes so the team can see workspace-size impact.
const start = performance.now();
const maps = await this.flatEntityMapsCacheService.getOrRecomputeManyOrAllFlatEntityMaps(args);
this.logger.debug(`flat map recompute took ${performance.now() - start}ms`);
```

**The "Aha" Moment:**

Performance work starts by finding shared paths, because shared paths multiply small costs.

**Socratic Checkpoint:**

1. Why is provider breadth a performance risk?
2. What would recreate Apollo clients?
3. Why might metadata recompute scale poorly?
4. What metric would you add first?
5. Which performance issue is most likely user-visible?

Self-grade rubric: strong answers discuss blast radius and measurement.

**Connects To:**

- Mission 22, because performance and security both audit shared boundaries.
- Mission 25, because ownership notes should include performance risks.

## Mission 22: The Security Audit

**Tier:** Senior  
**Time Estimate:** 75 minutes  
**Goal:** You will audit auth, authorization, input validation, CORS, JWT handling, and error exposure.

**The Concept:**

Security review asks: can the wrong person touch the wrong workspace, can untrusted input cross a boundary, and can errors leak more than they should?

**Design Intent (Before You Read the Code):**

1. Files involved: `main.ts:30-35`, `app.module.ts:114-146`, `rest-api-core.controller.ts:24-26`, `webhook.controller.ts:28-39`, `apollo.factory.ts:111-140`, `apollo.factory.ts:270-341`.
2. Auth headers, middleware context, guards, and filters form the security chain.
3. Pattern: frontend sends bearer/workspace/app version headers; backend hydrates context; guards authorize; filters shape errors.
4. Poor implementation exposes data across workspaces, allows permission bypass, or leaks sensitive internals.

**Find It In The Code:**

Risk 1: dynamic REST depends on guards.

```ts
// packages/twenty-server/src/engine/api/rest/core/controllers/rest-api-core.controller.ts:24-26
@Controller('rest')
@UseGuards(JwtAuthGuard, WorkspaceAuthGuard, CustomPermissionGuard)
@UseFilters(RestApiExceptionFilter)
export class RestApiCoreController {}
```

Risk 2: webhook settings are sensitive.

```ts
// packages/twenty-server/src/engine/metadata-modules/webhook/controllers/webhook.controller.ts:28-33
@Controller(['rest/webhooks', 'rest/metadata/webhooks'])
@UseGuards(
  JwtAuthGuard,
  WorkspaceAuthGuard,
  SettingsPermissionGuard(PermissionFlagType.API_KEYS_AND_WEBHOOKS),
)
```

Risk 3: frontend auth headers.

```ts
// packages/twenty-front/src/modules/apollo/services/apollo.factory.ts:126-138
const token = tokenPair.accessOrWorkspaceAgnosticToken?.token;
return {
  headers: {
    authorization: token ? `Bearer ${token}` : '',
    'x-locale': locale,
    ...(isDefined(this.currentWorkspace?.metadataVersion) && {
      'X-Schema-Version': `${this.currentWorkspace.metadataVersion}`,
    }),
    ...(this.appVersion && { 'X-App-Version': this.appVersion }),
  },
};
```

Fix style:

```ts
// Keep backend authorization decisive; frontend route hiding is only UX.
@UseGuards(JwtAuthGuard, WorkspaceAuthGuard, SettingsPermissionGuard(PermissionFlagType.API_KEYS_AND_WEBHOOKS))
```

Tests:

- Limited user cannot call `PATCH /rest/webhooks/:id`.
- Tokenless request to `/rest/*path` returns unauthorized.
- User with workspace A token cannot access workspace B record.
- Bad input produces a safe filtered error, not a stack trace.

**The "Aha" Moment:**

Frontend gates are courtesy; backend guards are authority.

**Socratic Checkpoint:**

1. Which backend decorator is most security-critical on dynamic REST?
2. Why are webhooks sensitive?
3. What does Apollo add to authenticate requests?
4. Why are filters part of security?
5. What regression test would you add first?

Self-grade rubric: strong answers distinguish authentication, authorization, input validation, and error shaping.

**Connects To:**

- Mission 23, because security risks need tests.
- Mission 25, because code review docs should list sensitive areas.

## Mission 23: Write the Test That Doesn't Exist

**Tier:** Senior  
**Time Estimate:** 60 minutes  
**Goal:** You will identify risky untested behavior and write one concrete regression test plan.

**The Concept:**

Tests are executable ownership notes. A good test says: "This behavior matters enough that future edits must prove they preserved it."

**Design Intent (Before You Read the Code):**

1. Files involved: `apollo.factory.ts:41-44`, `webhook.controller.ts:28-39`, `view-sort.service.ts:80-100`, existing test folders under `packages/twenty-front/src/modules/apollo/services/__tests__` and backend service test folders.
2. Risky behavior often crosses boundaries, making it harder to test.
3. Pattern: isolate the smallest boundary that proves the risk.
4. Poor test selection either tests implementation details only or misses the failure mode.

**Find It In The Code:**

Three risky behaviors:

1. Concurrent token renewal: `apollo.factory.ts:41-44` and `apollo.factory.ts:190-210`.
2. Webhook permission enforcement: `webhook.controller.ts:28-39`.
3. Metadata migration failure handling: `view-sort.service.ts:80-84`.

Runnable test sketch for token renewal:

```ts
// packages/twenty-front/src/modules/apollo/services/__tests__/apollo.factory.concurrent-renewal.test.ts
// Arrange two operations that both receive UNAUTHENTICATED.
// Mock renewToken and assert it is called once because renewalPromise is shared.
it('deduplicates concurrent token renewals', async () => {
  // mock getTokenPair to return an expired-but-present token pair
  // mock renewToken to resolve a fresh token pair
  // fire two Apollo operations that hit the error link concurrently
  // expect renewToken).toHaveBeenCalledTimes(1)
  // expect both operations).toEventuallyResolveOrRetry()
});
```

**The "Aha" Moment:**

The best missing test is usually at the boundary where a future refactor would be most tempting.

**Socratic Checkpoint:**

1. Why is concurrent token renewal risky?
2. Why test webhook permissions at API level?
3. What should a migration failure test assert?
4. What makes a test too implementation-specific?
5. Which test would you write first and why?

Self-grade rubric: strong answers choose high-impact behavior and explain the failure mode.

**Connects To:**

- Mission 24, because tests help define commit boundaries.
- Mission 25, because ownership notes should include testing gaps.

## Mission 24: The Git History Tells a Story

**Tier:** Senior  
**Time Estimate:** 45 minutes  
**Goal:** You will learn to shape commits so future engineers can understand why a change happened.

**The Concept:**

Commit history is the CRM's maintenance log. A good entry tells the next engineer what changed, why it changed, and what risk the reviewer checked.

**Design Intent (Before You Read the Code):**

1. Files involved: realistic changes across `apollo.factory.ts`, `webhook.controller.ts`, `view-sort.service.ts`, `AppRouterProviders.tsx`, and tests.
2. Each commit should map to one behavior.
3. Pattern: narrow intent, matching tests, generated files separated when noisy.
4. Poor commits mix refactor, behavior, generated output, and formatting.

**Find It In The Code:**

Example commits:

1. `fix(auth): dedupe concurrent Apollo token renewal`
   - Files: `packages/twenty-front/src/modules/apollo/services/apollo.factory.ts:41-44`, tests under Apollo factory.
   - Reviewer scrutinizes concurrency and no infinite retry.
2. `test(security): cover webhook permission enforcement`
   - Files: `packages/twenty-server/src/engine/metadata-modules/webhook/controllers/webhook.controller.ts:28-39`, integration tests.
   - Reviewer scrutinizes role fixture and forbidden response.
3. `perf(metadata): add timing around flat map recomputation`
   - Files: `view-sort.service.ts:87-100`, flat map cache service.
   - Reviewer scrutinizes metric cardinality and sensitive labels.
4. `refactor(front): split app router providers by runtime concern`
   - Files: `AppRouterProviders.tsx:43-96`, new provider group files.
   - Reviewer scrutinizes provider order.
5. `feat(views): support null ordering in view sorts`
   - Files: controller/input/service/generated frontend types/hook.
   - Reviewer scrutinizes contract alignment.

**The "Aha" Moment:**

A good commit should make the future debugging path shorter.

**Socratic Checkpoint:**

1. Which example should include generated files?
2. Which example should be split if it also reformats files?
3. What should reviewers focus on for provider refactor?
4. What makes a commit message useful six months later?
5. What does messy commit history reveal?

Self-grade rubric: strong answers connect commit shape to review risk and future debugging.

**Connects To:**

- Mission 25, because permanent docs are another form of future-maintenance aid.

