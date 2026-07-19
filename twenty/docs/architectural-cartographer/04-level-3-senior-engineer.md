# 04 - Level 3: Senior Engineer Guide

## 3.1 Architectural Critique

### What Is Done Well

Twenty has a clear boot chain: React entry at `packages/twenty-front/src/index.tsx:10`, app shell at `packages/twenty-front/src/modules/app/components/App.tsx:20`, route provider shell at `packages/twenty-front/src/modules/app/components/AppRouterProviders.tsx:44`, Nest entry at `packages/twenty-server/src/main.ts:30`, and backend module composition at `packages/twenty-server/src/app.module.ts:49`. That gives a new engineer a reliable top-down reading path.

The API boundary is centralized. Apollo auth, retry, token renewal, REST, stream, and upload behavior live in `packages/twenty-front/src/modules/apollo/services/apollo.factory.ts:98`. Backend GraphQL/REST/MCP middleware is centralized in `packages/twenty-server/src/app.module.ts:114`.

Metadata writes are disciplined. `ViewSortService` and `WebhookService` do not simply mutate metadata tables; they create flat entities and run workspace migration validation at `packages/twenty-server/src/engine/metadata-modules/view-sort/services/view-sort.service.ts:63` and `packages/twenty-server/src/engine/metadata-modules/webhook/webhook.service.ts:117`.

### Weak Points And Debt

The frontend provider stack is dense. `AppRouterProviders` nests many cross-cutting concerns before `<Outlet />` at `packages/twenty-front/src/modules/app/components/AppRouterProviders.tsx:44`, so boot regressions can look like route bugs, auth bugs, or metadata bugs at once.

Dynamic APIs reduce local readability. `RestApiCoreController` exposes wildcard routes like `@Get('*path')` at `packages/twenty-server/src/engine/api/rest/core/controllers/rest-api-core.controller.ts:83`, so a reviewer must inspect parsers and handlers to understand exact request shapes.

Generated types contain `any` for important scalars such as `JSON`, `JSONObject`, and `Upload` at `packages/twenty-front/src/generated-metadata/graphql.ts:16`. That is probably necessary in some places, but it makes runtime validation and DTO discipline more important.

### Ratings

| Dimension | Score | Justification |
|---|---:|---|
| Scalability | 4 | Metadata-driven design and workspace data source modules scale across dynamic objects; see `GlobalWorkspaceDataSourceModule` import at `app.module.ts:58`. Complexity scales too. |
| TypeScript discipline | 3 | Strong generated types and DTOs, but JSON/Upload `any` at `generated-metadata/graphql.ts:16` and wildcard REST reduce compile-time guarantees. |
| Separation of concerns | 4 | App shell, API transport, metadata services, and migration runner have distinct roles; provider density and dynamic routing make boundaries harder to see. |
| Testability | 3 | There are targeted tests and wrappers, but provider-heavy boot and metadata migrations require high-fidelity test setup. |
| Maintainability | 3 | The reading path is good, but a new engineer must learn generated types, flat maps, migrations, dynamic REST, Apollo links, and Jotai state before owning changes. |
| Security posture | 4 | Guards are consistently visible on controllers, such as webhook settings permission at `webhook.controller.ts:29`; risk lies in dynamic routes and guard regressions. |
| Performance | 3 | Lazy routes and centralized data clients help, but dense providers and metadata recompute paths can be expensive if over-triggered. |

## 3.2 Performance Audit

1. Provider-heavy route shell
   - Code: `packages/twenty-front/src/modules/app/components/AppRouterProviders.tsx:44`.
   - Risk: many effects mount above every page; accidental state changes can re-run broad parts of the app.
   - Fix direction: profile provider effects, memoize expensive context values, and split route-specific providers below route boundaries when they only serve a subset of pages.

2. Apollo client recreation dependencies
   - Code: `useApolloFactory` memoizes client construction at `packages/twenty-front/src/modules/apollo/hooks/useApolloFactory.ts:45`.
   - Risk: dependency mistakes can recreate Apollo clients, clearing cache and reissuing network calls.
   - Fix direction:
```ts
// Keep Apollo construction dependencies intentionally minimal.
// Add a dependency only if changing it must create a new client,
// otherwise update the factory through updateWorkspaceMember/updateCurrentWorkspace.
```

3. Metadata map recomputation after writes
   - Code: `ViewSortService.createOne` recomputes flat maps after migration at `packages/twenty-server/src/engine/metadata-modules/view-sort/services/view-sort.service.ts:87`.
   - Risk: repeated metadata writes could force expensive recomputes.
   - Fix direction: batch writes where possible and add instrumentation around `getOrRecomputeManyOrAllFlatEntityMaps`.

4. Raw cron/database queries
   - Code: workflow cron jobs use core data source raw queries at `packages/twenty-server/src/modules/workflow/workflow-trigger/automated-trigger/crons/jobs/workflow-cron-trigger-cron.job.ts:154`.
   - Risk: raw SQL paths can become slow without query plans and indexes.
   - Fix direction: add query plan tests/metrics for high-frequency cron jobs.

## 3.3 Security Audit

1. Dynamic REST wildcard routes
   - Code: `RestApiCoreController` at `packages/twenty-server/src/engine/api/rest/core/controllers/rest-api-core.controller.ts:24`.
   - Vector: path parsing or permission bugs could expose records across object boundaries.
   - Fix: keep `JwtAuthGuard`, `WorkspaceAuthGuard`, and `CustomPermissionGuard` at line 25; add regression tests for unauthorized object access.
   - Test: call `/rest/people` with a token lacking object read permission and assert forbidden.

2. Webhook management permissions
   - Code: `WebhookController` guards at `packages/twenty-server/src/engine/metadata-modules/webhook/controllers/webhook.controller.ts:29`.
   - Vector: unauthorized user could create outbound webhooks and exfiltrate workspace events.
   - Fix: never remove `SettingsPermissionGuard(PermissionFlagType.API_KEYS_AND_WEBHOOKS)`.
   - Test: user without the permission cannot `POST /rest/webhooks`.

3. Token renewal and sign-out behavior
   - Code: `ApolloFactory` handles unauthenticated errors at `packages/twenty-front/src/modules/apollo/services/apollo.factory.ts:270`.
   - Vector: bad renewal logic can leave stale tokens active or sign users out unexpectedly.
   - Fix: maintain shared `renewalPromise` dedupe at `packages/twenty-front/src/modules/apollo/services/apollo.factory.ts:41`.
   - Test: simulate concurrent `UNAUTHENTICATED` errors and assert one renewal request.

4. Upload size and file endpoints
   - Code: upload limits are configured at `packages/twenty-server/src/main.ts:74`.
   - Vector: oversized uploads can cause memory pressure or denial of service.
   - Fix: keep `settings.storage.maxFileSize` applied to JSON and upload paths.
   - Test: upload over max size returns 413 and frontend shows payload-too-large handling from `apollo.factory.ts:326`.

## 3.4 TypeScript Discipline Review

Strong typing:

- Generated GraphQL operations and fragments in `packages/twenty-front/src/generated-metadata/graphql.ts:1`.
- Narrowed current user state with `Pick<User, ...>` at `packages/twenty-front/src/modules/auth/states/currentUserState.ts:4`.
- Typed state factory in `packages/twenty-front/src/modules/ui/utilities/state/jotai/utils/createAtomState.ts:27`.
- Backend DTO returns in controllers such as `Promise<ViewSortDTO>` at `packages/twenty-server/src/engine/metadata-modules/view-sort/controllers/view-sort.controller.ts:92`.

Weak typing:

- `Scalars['JSON']`, `Scalars['JSONObject']`, and `Scalars['Upload']` are `any` in `packages/twenty-front/src/generated-metadata/graphql.ts:16`.
- Dynamic REST requests are typed as enriched requests, not domain-specific payloads, in `packages/twenty-server/src/engine/api/rest/core/controllers/rest-api-core.controller.ts:33`.
- Webhook REST update body is manually declared at `packages/twenty-server/src/engine/metadata-modules/webhook/controllers/webhook.controller.ts:69`, so it can drift from `UpdateWebhookInput`.

Purist first change: replace broad generated `any` consumption with runtime schemas at the boundary where JSON fields are read, not by editing generated files.

## 3.5 Custom Abstractions Inventory

| Abstraction | Hides | Keep/refactor/delete |
|---|---|---|
| `createAtomState` at `packages/twenty-front/src/modules/ui/utilities/state/jotai/utils/createAtomState.ts:27` | Jotai atom creation plus storage modes. | Keep; it standardizes state definitions. |
| `useApolloFactory` at `packages/twenty-front/src/modules/apollo/hooks/useApolloFactory.ts:24` | Apollo client creation tied to auth/workspace state. | Keep; document dependencies carefully. |
| `ApolloFactory` at `packages/twenty-front/src/modules/apollo/services/apollo.factory.ts:68` | Links, auth headers, renewal, retry, error handling. | Keep; split only if tests become painful. |
| `AppRouterProviders` at `packages/twenty-front/src/modules/app/components/AppRouterProviders.tsx:39` | Runtime provider orchestration. | Refactor over time by grouping providers into named shells. |
| `RestApiCoreService` at `packages/twenty-server/src/engine/api/rest/core/services/rest-api-core.service.ts:24` | One-vs-many dynamic object dispatch. | Keep; add docs/tests around path semantics. |
| `WorkspaceAuthContextMiddleware` attached at `packages/twenty-server/src/app.module.ts:118` | Workspace context hydration. | Keep; security critical. |
| Metadata services such as `ViewSortService` at `packages/twenty-server/src/engine/metadata-modules/view-sort/services/view-sort.service.ts:24` | Flat maps, migration validation, DTO conversion. | Keep; extract common create/update/delete metadata pattern only if duplication grows. |

## 3.6 Testing Assessment

What is tested:

- Frontend auth utilities and hooks have tests under `packages/twenty-front/src/modules/auth`.
- Apollo utilities have tests under `packages/twenty-front/src/modules/apollo/utils/__tests__`.
- Object-record query utilities have tests under `packages/twenty-front/src/modules/object-record/utils/__tests__`.
- Backend metadata/workflow services have tests such as `packages/twenty-server/src/engine/metadata-modules/object-permission/__tests__/object-permission.service.spec.ts` and workflow cron tests.

Hardest to test:

- Full provider boot from `AppRouterProviders` at `packages/twenty-front/src/modules/app/components/AppRouterProviders.tsx:44`.
- Metadata mutation plus workspace migration flow in `ViewSortService` at `packages/twenty-server/src/engine/metadata-modules/view-sort/services/view-sort.service.ts:63`.

Riskiest untested behaviors:

1. Concurrent token renewal dedupe in `ApolloFactory`.
2. Permission enforcement on dynamic REST wildcard object routes.
3. Metadata cache recompute after failed migration.

Example runnable test target: add an Apollo factory test for concurrent unauthenticated errors under `packages/twenty-front/src/modules/apollo/services/__tests__/apollo.factory.test.ts`, following the existing test file path found in the repo.

## 3.7 Bug Injection Exercise

Training-only bugs, not applied to the codebase:

1. Symptom: users are randomly redirected to sign-in during two simultaneous failed GraphQL requests. Test scenario: mock two `UNAUTHENTICATED` responses and assert only one token renewal occurs.
2. Symptom: a user without API/webhook permission can update a webhook. Test scenario: call `PATCH /rest/webhooks/:id` as a limited role and expect forbidden.
3. Symptom: view sort appears immediately but disappears after refresh. Test scenario: create a view sort, reload metadata, and assert it remains in flat view sort maps.
4. Symptom: uploading a large attachment hangs instead of showing a friendly error. Test scenario: upload above `settings.storage.maxFileSize` and assert 413 handling.
5. Symptom: switching locale updates text but date/number formatting stays old. Test scenario: load current user with a new locale and assert `initializeFormatPreferences` ran.

Debugging review prompt to paste later:

```text
I investigated the bugs. Here are my notes: [YOUR NOTES]

Evaluate my debugging approach:
1. Did I inspect the right files?
2. Did I form a good hypothesis?
3. What did I miss?
4. What would a senior engineer have checked first?
5. Walk me through the correct debugging path for each bug.
```

## 3.8 Git History Learning Exercise

1. `fix(auth): dedupe concurrent token renewal requests`
   - Files: `apollo.factory.ts`, Apollo factory tests.
   - Review focus: shared `renewalPromise`, unauthenticated handling, no infinite retry.
   - Ticket: AUTH-421.

2. `feat(metadata): allow descriptions on workspace webhooks`
   - Files: webhook DTOs, flat webhook utils, service, generated frontend types.
   - Review focus: migration validation and API compatibility.
   - Ticket: META-118.

3. `test(rest): cover forbidden access for dynamic object routes`
   - Files: REST integration tests, permission fixtures.
   - Review focus: object-level permission matrix.
   - Ticket: SEC-212.

4. `perf(metadata): instrument flat entity map recomputation`
   - Files: flat entity map cache service, metrics module.
   - Review focus: metric cardinality and no sensitive labels.
   - Ticket: PERF-083.

5. `refactor(front): group app router providers by concern`
   - Files: `AppRouterProviders.tsx`, new provider shells, provider tests.
   - Review focus: provider order remains behaviorally identical.
   - Ticket: FE-639.

Shape of history: the healthiest commits are narrow, explain a product or platform behavior, include tests where risk is high, and avoid mixing generated output with unrelated refactors.

## 3.9 If I Owned This Codebase

| Priority | Problem | Fix | Effort | Impact | First action |
|---:|---|---|---|---|---|
| 1 | Provider boot is hard to reason about. | Split `AppRouterProviders` into named provider groups with tests preserving order. | M | Maintainability | Snapshot current provider/effect order around `AppRouterProviders.tsx:44`. |
| 2 | Dynamic REST contract is hard to review. | Generate a REST route contract appendix from controllers and parsers. | M | Risk reduction | Start with `RestApiCoreController` and `RestApiCoreService`. |
| 3 | JSON `any` leaks through generated types. | Add runtime validators at JSON consumption boundaries. | M | Type/runtime safety | Inventory `Scalars['JSON']` usages from `generated-metadata/graphql.ts:16`. |
| 4 | Metadata migration flows are complex. | Create a contributor guide for metadata create/update/delete patterns. | S | Maintainability | Use `ViewSortService` and `WebhookService` as examples. |
| 5 | Permission-sensitive routes need strong regression coverage. | Add role-matrix integration tests for REST and metadata endpoints. | L | Security | Begin with webhook and dynamic object REST routes. |

