## Story 1: Clarify Sign-In Title
**Difficulty:** Easy
**Estimated Time:** 1.5 hours
**Skills You'll Practice:** React component edits, props, UI verification

**The Story:**
As a new user, I want clearer sign-in page headings so that I understand whether I am creating an account or logging in.

**Acceptance Criteria:**
- [ ] Sign-in title text is clearer and context-specific.
- [ ] Invite flow title remains correct.
- [ ] No regression in animated title behavior.

**Files You'll Likely Touch:**
- `packages/twenty-front/src/modules/auth/components/Title.tsx` - title presentation component.
- `packages/twenty-front/src/pages/auth/SignInUp.tsx` - title usage context.

**High-Level Implementation Plan:**
1. Inspect where `Title` is rendered in auth page.
2. Update title strings by auth mode.
3. Verify animation flag behavior still works.

**Tips:**
- Reuse existing `animate` and `noMarginTop` props.
- Keep text changes inside route context, not global component defaults.
- Check invite route branch in router (`useCreateAppRouter.tsx:133-138`).

**What Could Go Wrong:**
- Title changes all auth screens unintentionally.
- Invite flow shows wrong title.

**Stretch Goal:**
Add localization keys instead of hardcoded text.

**Connects To:**
Story 2, because it introduces typed state-driven UI branching.

## Story 2: Show Workspace Domain Hint on Auth
**Difficulty:** Easy
**Estimated Time:** 2 hours
**Skills You'll Practice:** State read, domain context, conditional rendering

**The Story:**
As a user, I want to see which workspace domain I am authenticating into.

**Acceptance Criteria:**
- [ ] Domain hint renders when workspace URL context exists.
- [ ] Hint is hidden on global auth route.
- [ ] No redirect logic changes.

**Files You'll Likely Touch:**
- `packages/twenty-front/src/modules/workspace/components/WorkspaceProviderEffect.tsx`
- `packages/twenty-front/src/pages/auth/SignInUp.tsx`

**High-Level Implementation Plan:**
1. Read current location/workspace domain hooks.
2. Add display-only domain hint prop.
3. Render hint with safe fallback.

**Tips:**
- Do not mutate redirect behavior.
- Keep hint derivation read-only.
- Verify multi-workspace guard conditions.

**What Could Go Wrong:**
- Leaking wrong domain in default-domain contexts.
- Triggering rerenders via derived URL parsing in render loop.

**Stretch Goal:**
Add tooltip explaining workspace-specific login.

**Connects To:**
Story 3.

## Story 3: Surface Current User Onboarding Status in Settings Header
**Difficulty:** Easy
**Estimated Time:** 2 hours
**Skills You'll Practice:** Jotai state consumption, typed user fields

**The Story:**
As a workspace member, I want to see my onboarding status in settings.

**Acceptance Criteria:**
- [ ] Status renders from `currentUserState`.
- [ ] Empty state handled safely.
- [ ] No network call added.

**Files You'll Likely Touch:**
- `packages/twenty-front/src/modules/auth/states/currentUserState.ts`
- `packages/twenty-front/src/pages/settings/SettingsProfile.tsx`

**High-Level Implementation Plan:**
1. Read onboarding field from `CurrentUser` type.
2. Display status badge in profile page header.
3. Validate null user behavior.

**Tips:**
- Keep rendering resilient when user is null.
- Use existing style primitives.
- Avoid adding duplicate status mapping logic.

**What Could Go Wrong:**
- Accessing undefined user in render.
- Mislabeling enum values.

**Stretch Goal:**
Add mapping helper for human-readable status labels.

**Connects To:**
Story 4.

## Story 4: Add a View Sort Count Indicator
**Difficulty:** Medium
**Estimated Time:** 4 hours
**Skills You'll Practice:** GraphQL query usage, UI composition, metadata feature reading

**The Story:**
As a user managing a view, I want to see how many sort rules are active.

**Acceptance Criteria:**
- [ ] Count reflects current view sort list length.
- [ ] Updates after create/update/delete sort mutations.
- [ ] Empty state displays 0.

**Files You'll Likely Touch:**
- `packages/twenty-front/src/modules/views/hooks/internal/usePerformViewSortAPIPersist.ts`
- `packages/twenty-front/src/modules/views/...` (view settings UI component)

**High-Level Implementation Plan:**
1. Locate view sort list query consumer.
2. Add derived count display.
3. Ensure mutation completions refresh count source.

**Tips:**
- Reuse existing metadata query cache updates.
- Keep count derived, not separately stored.
- Verify delete and destroy semantics.

**What Could Go Wrong:**
- Stale count due to cache mismatch.
- Counting soft-deleted sorts.

**Stretch Goal:**
Add tooltip listing sorted fields.

**Connects To:**
Story 5.

## Story 5: Add Filter to View Sort REST List by Direction
**Difficulty:** Medium-Easy
**Estimated Time:** 4 hours
**Skills You'll Practice:** Controller query params, service filtering, DTO behavior

**The Story:**
As an admin, I want to filter view sorts by direction so I can audit sort consistency.

**Acceptance Criteria:**
- [ ] `GET /rest/metadata/viewSorts?direction=ASC|DESC` supported.
- [ ] Invalid direction returns controlled error.
- [ ] Existing behavior unchanged when param absent.

**Files You'll Likely Touch:**
- `packages/twenty-server/src/engine/metadata-modules/view-sort/controllers/view-sort.controller.ts`
- `packages/twenty-server/src/engine/metadata-modules/view-sort/services/view-sort.service.ts`
- `packages/twenty-shared/src/types/ViewSortDirection.ts`

**High-Level Implementation Plan:**
1. Add optional query param in controller.
2. Add service filter branch.
3. Reuse enum validation.

**Tips:**
- Keep `deletedAt: IsNull()` invariant.
- Do not weaken existing guards.
- Preserve relation loading behavior.

**What Could Go Wrong:**
- Filtering at controller level only, causing unbounded service behavior.
- Accepting invalid direction string silently.

**Stretch Goal:**
Mirror filter in GraphQL `getViewSorts` query args.

**Connects To:**
Story 6.

## Story 6: Add New API Endpoint to Duplicate View Sort Set
**Difficulty:** Medium
**Estimated Time:** 6 hours
**Skills You'll Practice:** New controller route, service orchestration, permissions

**The Story:**
As a user, I want to duplicate all sort rules from one view to another.

**Acceptance Criteria:**
- [ ] New endpoint accepts source and target view IDs.
- [ ] Permission guards enforced.
- [ ] Duplicate conflicts are handled predictably.

**Files You'll Likely Touch:**
- `view-sort.controller.ts`
- `view-sort.service.ts`
- `create-view-sort.input.ts` or new DTO

**High-Level Implementation Plan:**
1. Define endpoint + input DTO.
2. Fetch source sorts, map to create payloads.
3. Use existing migration-driven create path.

**Tips:**
- Reuse `createOne` semantics rather than bypassing validation.
- Consider unique constraint `fieldMetadataId + viewId`.
- Return structured result for partial failures.

**What Could Go Wrong:**
- Attempting raw inserts bypassing metadata map consistency.
- Copying deleted sorts.

**Stretch Goal:**
Expose feature via frontend view settings action.

**Connects To:**
Story 7.

## Story 7: Add Frontend UI Action for Duplicate Sort Set
**Difficulty:** Medium
**Estimated Time:** 6 hours
**Skills You'll Practice:** UI action wiring, API integration, optimistic UX

**The Story:**
As a power user, I want a UI action to copy sort rules from one view to another.

**Acceptance Criteria:**
- [ ] Action appears in view configuration UI.
- [ ] Calls new backend endpoint.
- [ ] Success and error snackbars shown.

**Files You'll Likely Touch:**
- view configuration component in `packages/twenty-front/src/modules/views/...`
- API hook layer near `usePerformViewSortAPIPersist.ts`
- snackbar/error utilities

**High-Level Implementation Plan:**
1. Add action trigger in view settings UI.
2. Create hook wrapper for endpoint call.
3. Refresh view sort list on success.

**Tips:**
- Keep failure path explicit.
- Reuse existing `enqueueErrorSnackBar` patterns.
- Ensure domain/workspace context preserved in request.

**What Could Go Wrong:**
- UI success without backend success.
- Missing refresh after mutation.

**Stretch Goal:**
Add undo flow by tracking created sort IDs.

**Connects To:**
Story 8.

## Story 8: Add Audit Log Event for View Sort Updates
**Difficulty:** Hard
**Estimated Time:** 8 hours
**Skills You'll Practice:** Cross-layer instrumentation, backend event logging, observability

**The Story:**
As a security-conscious admin, I want view sort modifications captured in audit logs.

**Acceptance Criteria:**
- [ ] Create/update/delete actions emit audit events.
- [ ] Event includes workspace, actor, and target view sort ID.
- [ ] Event visible in existing event log UI/query path.

**Files You'll Likely Touch:**
- `view-sort.service.ts`
- event logs service/module files under `engine/core-modules/event-logs`
- frontend event logs query/table components

**High-Level Implementation Plan:**
1. Identify existing event logging API.
2. Emit events in mutation paths.
3. Confirm frontend table can display new event type.

**Tips:**
- Keep event payload small and structured.
- Avoid logging sensitive fields.
- Reuse existing event enums where possible.

**What Could Go Wrong:**
- Logging on success path only but missing rollback scenarios.
- Excessive payload size in log record.

**Stretch Goal:**
Add filter chip for view-sort event type.

**Connects To:**
Story 9.

## Story 9: Add Workspace-Level Permission Gate for View Sort Mutation
**Difficulty:** Hard
**Estimated Time:** 10 hours
**Skills You'll Practice:** Authorization model changes, guard updates, frontend permission gating

**The Story:**
As a workspace admin, I want only authorized roles to modify view sorts.

**Acceptance Criteria:**
- [ ] New permission flag checked in backend guards.
- [ ] Unauthorized users receive consistent forbidden response.
- [ ] Frontend hides mutation controls when permission absent.

**Files You'll Likely Touch:**
- view permission guards in `packages/twenty-server/src/engine/metadata-modules/view-permissions/guards/...`
- user/workspace permission state usage in frontend
- affected UI components in views/settings

**High-Level Implementation Plan:**
1. Introduce or reuse permission flag.
2. Enforce in guard chain.
3. Mirror in frontend control visibility.

**Tips:**
- Backend enforcement is source of truth.
- Keep error semantics consistent across REST/GraphQL.
- Reuse `currentUserWorkspace` permission flags.

**What Could Go Wrong:**
- UI hides controls but backend still allows mutation.
- Guard checks wrong permission scope.

**Stretch Goal:**
Add admin panel toggle for this permission.

**Connects To:**
Story 10.

## Story 10: Metadata Mutation Throughput and Safety Upgrade
**Difficulty:** Expert
**Estimated Time:** 14 hours
**Skills You'll Practice:** Architecture tradeoff analysis, performance tuning, consistency guarantees

**The Story:**
As a platform maintainer, I want metadata mutations to be faster without sacrificing validation safety.

**Acceptance Criteria:**
- [ ] Proposal and implementation for targeted flat-map invalidation exists.
- [ ] No behavior regression in create/update/delete view sort flows.
- [ ] Added tests or instrumentation proving correctness.

**Files You'll Likely Touch:**
- `view-sort.service.ts`
- flat entity map cache services under `engine/metadata-modules/flat-entity/services`
- migration validate/build/run service usage points

**High-Level Implementation Plan:**
1. Baseline current recompute flow and hotspots.
2. Design targeted invalidation strategy.
3. Implement incrementally behind feature flag or guardrail.
4. Add observability for cache hit/recompute behavior.

**Tips:**
- Protect correctness first, then optimize.
- Keep fallback to full recompute path.
- Write explicit invariants in code comments.

**What Could Go Wrong:**
- Stale metadata maps producing phantom UI behavior.
- Optimization bypasses validation guarantees.

**Stretch Goal:**
Generalize optimization pattern for view filters and groups.

**Connects To:**
No next story. This is an ownership-level capstone.
