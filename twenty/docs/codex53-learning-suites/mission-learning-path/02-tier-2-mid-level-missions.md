# Tier 2 Mid-Level Missions

### Mission 9: State Has a Home and a Reason
**Tier:** Mid-Level
**Time Estimate:** 45 minutes
**Goal:** Explain why specific state lives in local/component/global layers.
**The Concept:** State ownership follows read/write radius.
**Design Intent Before You Read the Code:** compare `currentUserState`, `currentWorkspaceState`, provider effects.
**Find It In The Code:** `currentUserState.ts:4-20`, `useLoadCurrentUser.ts:65-127`, `AppRouterProviders.tsx:44-95`.
**The Aha Moment:** Wrong state placement creates accidental coupling.
**Socratic Checkpoint:** 5 questions on state locality, ownership, side effects, and failure modes.
**How to self-grade:** prove each state location with one consumer and one writer.
**Connects To:** Mission 10, 14.

### Mission 10: The Custom Hook Ecosystem
**Tier:** Mid-Level
**Time Estimate:** 40 minutes
**Goal:** Understand how hooks hide orchestration complexity.
**The Concept:** Hooks are reusable behavior containers.
**Design Intent Before You Read the Code:** use mutation persistence hook.
**Find It In The Code:** `usePerformViewSortAPIPersist.ts:21-223`.
**The Aha Moment:** One hook can centralize error semantics for a whole feature family.
**Socratic Checkpoint:** 5 questions.
**How to self-grade:** explain callback dependencies and error branches.
**Connects To:** Mission 14, 15.

### Mission 11: Side Effects Are Promises to the System
**Tier:** Mid-Level
**Time Estimate:** 35 minutes
**Goal:** Analyze dependency arrays and side-effect safety.
**The Concept:** `useEffect` is a contract with time and dependencies.
**Design Intent Before You Read the Code:** workspace redirect effects.
**Find It In The Code:** `WorkspaceProviderEffect.tsx:44-87`.
**The Aha Moment:** Missing one dependency can become cross-workspace bugs.
**Socratic Checkpoint:** 5 questions.
**How to self-grade:** explain trigger and cleanup logic.
**Connects To:** Mission 14.

### Mission 12: The Full API Contract
**Tier:** Mid-Level
**Time Estimate:** 60 minutes
**Goal:** Map high-value backend surfaces and frontend call sites.
**The Concept:** API contract is a behavioral promise.
**Design Intent Before You Read the Code:** inspect guarded REST + metadata GraphQL patterns.
**Find It In The Code:** `dashboard.controller.ts:11-24`, `view-sort.controller.ts:38-124`, `view-sort.resolver.ts:23-103`, `apollo.factory.ts:98-160`.
**The Aha Moment:** Contract quality is measured by consistency across transport layers.
**Socratic Checkpoint:** 5 questions.
**How to self-grade:** include auth, request/response, and error behavior per endpoint family.
**Connects To:** Mission 13, 15.

### Mission 13: The Middleware Chain
**Tier:** Mid-Level
**Time Estimate:** 40 minutes
**Goal:** Trace authenticated request path through middleware.
**The Concept:** Middleware is progressive context enrichment.
**Design Intent Before You Read the Code:** inspect module `configure` and bootstrap middleware.
**Find It In The Code:** `main.ts:56-88`, `app.module.ts:114-145`.
**The Aha Moment:** Order is behavior.
**Socratic Checkpoint:** 5 questions.
**How to self-grade:** show exact order and one breakage from reorder.
**Connects To:** Mission 14, 22.

### Mission 14: End-to-End Feature Trace
**Tier:** Mid-Level
**Time Estimate:** 75 minutes
**Goal:** Trace view sort mutation from UI to persistence and back.
**The Concept:** Full-stack tracing builds debugging intuition.
**Design Intent Before You Read the Code:** use view sort as representative metadata write.
**Find It In The Code:** `usePerformViewSortAPIPersist.ts:30-223`, `view-sort.resolver.ts:52-102`, `view-sort.controller.ts:87-124`, `view-sort.service.ts:34-241`, `view-sort.entity.ts:19-72`.
**The Aha Moment:** Writes are validated through metadata migration, not raw CRUD.
**Socratic Checkpoint:** 5 questions.
**How to self-grade:** explain what breaks at each missing step.
**Connects To:** Mission 19, 21.

### Mission 15: Read the Diff Like an Engineer
**Tier:** Mid-Level
**Time Estimate:** 50 minutes
**Goal:** Practice PR review strategy on a realistic change.
**The Concept:** Good diff reading is risk triage.
**Design Intent Before You Read the Code:** hypothetical `subFieldName` edit support.
**Find It In The Code:** entity + input + service + frontend mutation variable flow.
**The Aha Moment:** Read contracts first, implementation second.
**Socratic Checkpoint:** 5 quiz questions.
**How to self-grade:** prioritize correctness, auth, and data integrity checks.
**Connects To:** Mission 24.

### Mission 16: Composition Over Inheritance
**Tier:** Mid-Level
**Time Estimate:** 35 minutes
**Goal:** Explain compositional design in provider stack.
**The Concept:** Composition builds behavior by layering.
**Design Intent Before You Read the Code:** provider stack as composed runtime.
**Find It In The Code:** `AppRouterProviders.tsx:44-95`.
**The Aha Moment:** Composition scales features but can hide dependency coupling.
**Socratic Checkpoint:** 5 questions.
**How to self-grade:** identify two compositional benefits and two costs.
**Connects To:** Mission 18.

### Mission 17: TypeScript's Hidden Work
**Tier:** Mid-Level
**Time Estimate:** 40 minutes
**Goal:** Identify non-obvious type safety wins.
**The Concept:** Type systems prevent invalid state transitions.
**Design Intent Before You Read the Code:** inspect generated mutation vars + enums + picked state types.
**Find It In The Code:** `usePerformViewSortAPIPersist.ts:11-19`, `currentUserState.ts:4-20`, `view-sort.entity.ts:44-50`.
**The Aha Moment:** Best TS usage removes entire categories of reviewer comments.
**Socratic Checkpoint:** 5 questions.
**How to self-grade:** include one runtime bug that each type layer prevents.
**Connects To:** Mission 18, 19.
