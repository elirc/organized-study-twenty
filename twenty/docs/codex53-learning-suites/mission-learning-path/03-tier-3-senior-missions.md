# Tier 3 Senior Missions

### Mission 18: Reverse-Engineer the Architecture Decisions
**Tier:** Senior
**Time Estimate:** 70 minutes
**Goal:** Infer why this architecture exists, not just what it is.
**The Concept:** Architecture is optimized compromise.
**Design Intent Before You Read the Code:** module composition, provider layering, migration-driven writes.
**Find It In The Code:** `app.module.ts:49-146`, `AppRouterProviders.tsx:44-95`, `view-sort.service.ts:63-85`.
**The Aha Moment:** Constraints, not taste, explain most structure.
**Socratic Checkpoint:** 5 questions + rubric.
**Connects To:** Mission 19, 24.

### Mission 19: Find the Bugs Before They Happen
**Tier:** Senior
**Time Estimate:** 60 minutes
**Goal:** Identify latent failure patterns before production incidents.
**The Concept:** Preventive debugging.
**Design Intent Before You Read the Code:** inspect side effects, auth renewal, migration write semantics.
**Find It In The Code:** `WorkspaceProviderEffect.tsx:44-87`, `apollo.factory.ts:190-210`, `view-sort.service.ts:200-241`.
**The Aha Moment:** Most outages are predictable from weak invariants.
**Socratic Checkpoint:** 5 questions + rubric.
**Connects To:** Mission 21, 22.

### Mission 20: The Bug Injection Challenge
**Tier:** Senior
**Time Estimate:** 45 minutes
**Goal:** Train symptom-first debugging.
**The Concept:** Hypothesis quality beats random file browsing.
**Design Intent Before You Read the Code:** use provided symptom scenarios only.
**Find It In The Code:** candidate areas listed, no code changes required.
**The Aha Moment:** Start with behavior, then narrow by contract boundaries.
**Socratic Checkpoint:** 5 questions + rubric.
**Connects To:** Mission 23.

### Mission 21: Performance X-Ray
**Tier:** Senior
**Time Estimate:** 75 minutes
**Goal:** Find and prioritize top performance risks.
**The Concept:** Performance work is bottleneck triage.
**Design Intent Before You Read the Code:** provider rerender risk + metadata recompute cost.
**Find It In The Code:** `AppRouterProviders.tsx:44-95`, `view-sort.service.ts:87-93`, `155-161`, `226-232`.
**The Aha Moment:** The slow part is often orchestration, not SQL alone.
**Socratic Checkpoint:** 5 questions + rubric.
**Connects To:** Mission 25.

### Mission 22: The Security Audit
**Tier:** Senior
**Time Estimate:** 75 minutes
**Goal:** Evaluate authz/authn and input safety posture.
**The Concept:** Security is layered denial of unsafe states.
**Design Intent Before You Read the Code:** guard chains, proxy trust, upload paths.
**Find It In The Code:** `dashboard.controller.ts:12`, `view-sort.controller.ts:39-44`, `main.ts:49-55`, `74-88`.
**The Aha Moment:** Most security bugs are missing checks at boundaries, not cryptography failures.
**Socratic Checkpoint:** 5 questions + rubric.
**Connects To:** Mission 25.

### Mission 23: Write the Test That Doesn't Exist
**Tier:** Senior
**Time Estimate:** 60 minutes
**Goal:** Add high-value coverage where architecture is fragile.
**The Concept:** Tests should protect invariants and contracts.
**Design Intent Before You Read the Code:** choose one risky untested behavior from mission 19.
**Find It In The Code:** `view-sort.controller.ts`, `view-sort.service.ts`, relevant guard/validator.
**The Aha Moment:** Great tests lock behavioral contracts, not implementation details.
**Socratic Checkpoint:** 5 questions + rubric.
**Connects To:** Mission 25.

### Mission 24: The Git History Tells a Story
**Tier:** Senior
**Time Estimate:** 40 minutes
**Goal:** Infer team habits from commit structure and scope.
**The Concept:** Commit history is organizational telemetry.
**Design Intent Before You Read the Code:** analyze realistic commit messages against architecture hotspots.
**Find It In The Code:** focus on files in previous missions.
**The Aha Moment:** Healthy history isolates risk and preserves intent.
**Socratic Checkpoint:** 5 questions + rubric.
**Connects To:** Mission 25.
