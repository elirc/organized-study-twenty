# Mission 25 Reference Artifacts

## Doc 1: Junior Onboarding Checklist
- Clone/install/start.
- Verify frontend root render and backend health boot.
- Trace one guarded route and one user-state hydration path.
- First PR checklist: touched file ownership, guard impact, state impact, test impact.

## Doc 2: Architecture Guide for New Engineers
- Route tree (`useCreateAppRouter.tsx:109-267`).
- Provider runtime (`AppRouterProviders.tsx:44-95`).
- Backend middleware chain (`app.module.ts:114-145`).
- Metadata write path (`view-sort.service.ts:34-241`).

## Doc 3: Code Review Checklist
1. Contract/type changes first.
2. Auth/guard consistency.
3. State fan-out correctness.
4. Error path behavior.
5. Soft delete semantics.

## Doc 4: Debugging Playbook
- Symptom: cross-workspace redirect loop -> `WorkspaceProviderEffect.tsx:44-87`.
- Symptom: stale user permission -> `useLoadCurrentUser.ts:75-85`.
- Symptom: metadata write conflict -> `view-sort.service.ts` validation branches.

## Doc 5: Change Playbook
- Branch -> implement -> lint/test target -> PR narrative -> reviewer focus list.
- Include impacted files and rollback strategy before merge.

## Doc 6: Senior Ownership Notes
- Monitor: auth renewal failures, migration validation failures, provider performance drift.
- Improve: observability around token refresh and metadata recompute.
- Leave alone for now: stable boot wiring unless feature requires order changes.

## Doc 7: Interview Walkthrough
- Non-technical: customizable open-source CRM platform.
- Technical: React/Jotai/Apollo frontend + Nest modular backend + metadata-managed writes.
- Full-stack example: view sort mutation lifecycle.
- Tradeoff: safety and flexibility vs complexity and onboarding burden.
