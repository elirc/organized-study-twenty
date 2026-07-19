# Validation, Auth & Permissions

## The full validation gauntlet (write path)

A create request passes, in order:

| # | Layer | What it checks | Anchor |
| --- | --- | --- | --- |
| 1 | Front sanitize | strips non-persistable fields from input | [useCreateOneRecord.ts#L89-L95](../../../packages/twenty-front/src/modules/object-record/hooks/useCreateOneRecord.ts#L89-L95) |
| 2 | GraphQL schema | field existence/types per generated SDL | [workspace-schema.factory.ts#L57-L65](../../../packages/twenty-server/src/engine/api/graphql/workspace-schema.factory.ts#L57-L65) |
| 3 | Runner `validate()` | operation-specific invariants: UUID validity, no mutations on remote objects | [common-create-one-query-runner.service.ts#L92-L103](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-create-one-query-runner.service.ts#L92-L103) |
| 4 | Arg processors | per-field-type coercion/validation via metadata | invoked at [common-create-one-query-runner.service.ts#L62-L68](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-create-one-query-runner.service.ts#L62-L68) |
| 5 | Query complexity | rejects nested 1:N-in-1:N; caps relation fan-out | [common-base-query-runner.service.ts#L397-L431](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-base-query-runner.service.ts#L397-L431) |
| 6 | ORM permission validation | object/field grants + row-level predicates on the final expression tree | [workspace-select-query-builder.ts#L342-L352](../../../packages/twenty-server/src/engine/twenty-orm/repository/workspace-select-query-builder.ts#L342-L352), [workspace-insert-query-builder.ts#L222](../../../packages/twenty-server/src/engine/twenty-orm/repository/workspace-insert-query-builder.ts#L222) |

Front validation is UX; server validation is law. The client sanitizer (1) exists so users get instant feedback — a hostile client skips it, which is why 2–6 exist.

## Authentication: who are you?

Token → context in one place: [jwt-auth.guard.ts#L23-L49](../../../packages/twenty-server/src/engine/guards/jwt-auth.guard.ts#L23-L49). Three principal types share one context shape — **user** (userWorkspaceId), **API key**, **application** — rejected unless at least one is present (L35-L45). SSO diversity (Google, Microsoft, OIDC, SAML) is quarantined in per-provider guards under [auth/guards/](../../../packages/twenty-server/src/engine/core-modules/auth/guards). Background jobs have no caller, so they *fabricate* a system principal: `buildSystemAuthContext(workspaceId)` ([run-workflow.job.ts#L47](../../../packages/twenty-server/src/modules/workflow/workflow-runner/jobs/run-workflow.job.ts#L47)) — every codebase you'll ever work in has this problem; remember the shape.

## Authorization: what may you do?

Distinct mechanisms, layered (defense in depth):

1. **Settings permissions (capabilities):** admin-ish flags checked in the pipeline for system objects ([common-base-query-runner.service.ts#L274-L311](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-base-query-runner.service.ts#L274-L311)), and via `SettingsPermissionGuard` on metadata mutations ([engine/guards/settings-permission.guard.ts](../../../packages/twenty-server/src/engine/guards/settings-permission.guard.ts)).
2. **Object permissions (CRUD grants per role per object):** resolved into a `rolePermissionConfig` and handed to the repository ([common-base-query-runner.service.ts#L313-L348](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-base-query-runner.service.ts#L313-L348)); enforced when queries execute, including **joins** — you can't smuggle a forbidden object in through a relation ([permissions.utils.ts#L279-L360](../../../packages/twenty-server/src/engine/twenty-orm/repository/permissions.utils.ts#L279-L360)).
3. **Row-level permission predicates:** predicates appended to the query itself ([workspace-select-query-builder.ts#L369-L390](../../../packages/twenty-server/src/engine/twenty-orm/repository/workspace-select-query-builder.ts#L369-L390)) — filtering, not post-hoc checking, so pagination and counts stay correct.
4. **Field-level restriction:** restricted fields stripped from selects (see `restrictedFields` handling in [permissions.utils.ts](../../../packages/twenty-server/src/engine/twenty-orm/repository/permissions.utils.ts)).

Tenant isolation itself is *structural*: each workspace's records live in a separate Postgres schema, and the datasource is resolved from the authenticated workspace context ([common-base-query-runner.service.ts#L144-L153](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-base-query-runner.service.ts#L144-L153)) — classic IDOR ("change the id in the URL") can't cross tenants because the *connection* is already scoped. Within a workspace, cross-user leakage is what layers 2–4 prevent.

## What a junior misses vs what a senior checks

| Junior misses | Senior checks |
| --- | --- |
| Auth ≠ authz — "logged in" isn't "allowed" | Which of the four layers covers each new endpoint |
| API keys and system jobs also hit your code | Every `shouldBypassPermissionChecks` call site (grep it) |
| Permissions on the object you *return*, not just the one you query | Join/relation permission behavior (layer 2 handles it — verify for new query shapes) |
| Error text leaking policy detail | Errors are standardized (`STANDARD_ERROR_MESSAGE`, [common-base-query-runner.service.ts#L328](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-base-query-runner.service.ts#L328)) — deny without explaining internals |
| Rate limits | Only API-key traffic is token-bucket throttled in the pipeline ([L350-L395](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-base-query-runner.service.ts#L350-L395)); know what covers browsers (captcha module, session limits — *investigate* before asserting coverage) |

## Drills

1. You add a new resolver that returns records with a relation to `person`. List which of the four authz layers protect the related records and where you'd write the test proving it. Strong: your test is a *cross-role* integration test, not a unit mock.
2. Grep `shouldBypassPermissionChecks` across `twenty-orm`. For three call sites, write one sentence each justifying (or challenging) the bypass. Strong: you identify the common pattern (system-context reads for event payloads) and state the invariant that keeps it safe ("bypass output must never flow to a user response unfiltered" — verify it holds).

## Interview angle

- "Walk me through auth in an app you know" — do it in the order above: authenticate once, bind to request, authorize in layers, enforce at the data layer. ([08-interview-prep/03](../08-interview-prep/03-api-and-data-modeling-questions.md) Q7–Q9)
- "How do you prevent IDOR?" — structural scoping first (connection-level), grants second, row predicates third. That ordering signals seniority.
