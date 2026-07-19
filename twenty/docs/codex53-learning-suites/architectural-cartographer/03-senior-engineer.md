# 03 Senior Engineer

## Architectural critique (1-5)
- Scalability: 4/5. Module decomposition is strong (`app.module.ts:49-79`), but provider and middleware depth increases cognitive load.
- TypeScript discipline: 4/5. Strong generated types and shared enums; selective casts remain (`useLoadCurrentUser.ts:81-83`).
- Separation of concerns: 4/5. Controller/service/entity separation is clear; composition complexity is high in frontend provider stack.
- Testability: 3.5/5. Many tests exist across repo, but integration-heavy flows need more cross-layer tests.
- Maintainability: 3.5/5. Large monorepo and many feature surfaces require strong onboarding docs and conventions.
- Security posture: 4/5. Guard-heavy backend and auth middleware layering are solid (`view-sort.controller.ts:39-44`, `dashboard.controller.ts:12`).
- Performance: 3.5/5. Potential over-render risk in deep provider trees; migration-based writes can be heavier than direct updates.

## Performance audit (top findings)
1. Provider tree depth can trigger avoidable rerenders (`AppRouterProviders.tsx:44-95`).
Fix pattern: memoize stable values for context providers and split high-churn effects.
2. Token renewal retries may hold stalled pipelines under repeated auth failures (`apollo.factory.ts:190-210`).
Fix pattern: bounded circuit-breaker per session.
3. Recompute-many flat maps after each metadata write adds overhead (`view-sort.service.ts:87-93`, `155-161`, `226-232`).
Fix pattern: targeted invalidation by metadata namespace.

## Security audit (top findings)
1. Ensure no unguarded REST metadata endpoint variants outside expected modules.
Audit baseline: `view-sort.controller.ts:39-44`, `87-114`.
2. Validate all upload surfaces retain size limits and path restrictions (`main.ts:74-88`).
3. Review header trust and proxy config behavior in multi-proxy deployments (`main.ts:49-55`).

## TypeScript discipline review
Strong:
- Shared enums and generated mutation variable types (`usePerformViewSortAPIPersist.ts:11-19`).
- Typed state contracts (`currentUserState.ts:4-20`).
Weak:
- Casting permissions payload in user hydration (`useLoadCurrentUser.ts:81-83`).

## Custom abstractions inventory (selected)
- `ApolloFactory` handles auth context, retries, Sentry, REST/GraphQL link composition (`apollo.factory.ts:68-405`). Keep, but consider splitting renewal, telemetry, and link assembly.
- `WorkspaceProviderEffect` centralizes workspace domain redirection logic (`WorkspaceProviderEffect.tsx:44-87`). Keep.
- `ViewSortService` encapsulates metadata migration workflow (`view-sort.service.ts:34-301`). Keep, but document more aggressively.

## Testing assessment
Untested high-risk behaviors:
1. Concurrent token renewal race handling.
2. Guard/filter parity between REST and GraphQL metadata surfaces.
3. Migration-driven metadata update rollback semantics.

Runnable test candidate (outline): add integration spec validating `PATCH /rest/metadata/viewSorts/:id` rejects invalid direction and preserves existing direction.
Primary files: `view-sort.controller.ts`, `view-sort.service.ts`, `flat-view-sort-validator.service.ts`.

## Bug injection exercise (symptoms only)
1. Sorting changes appear saved but vanish after refresh.
Scenario: update succeeds in UI toast, backend returns stale map.
2. Unauthorized user can duplicate dashboard in one workspace role combo.
3. App loops between domains on login for multi-workspace users.
4. Upload of medium files intermittently fails with generic errors.
5. Settings route visible to user lacking required permission flags.

## Git history learning exercise (sample commits)
1. `feat(metadata): add subFieldName to view sort update pipeline`
2. `fix(auth): deduplicate token renewal retries across parallel operations`
3. `refactor(front): split AppRouterProviders into domain-specific provider groups`
4. `test(view-sort): add integration coverage for soft delete and destroy flows`
5. `chore(config): tighten env validation for Redis and ClickHouse optional mode`

## If I owned this codebase: first 5 changes
1. Provider tree modularization (M, maintainability high).
2. Auth renewal observability + circuit breaker (M, risk reduction high).
3. Metadata write-path performance instrumentation (M, performance medium-high).
4. Guard/filter parity audit automation (S, security/reliability high).
5. End-to-end onboarding test scripts for local setup (M, contributor velocity high).
