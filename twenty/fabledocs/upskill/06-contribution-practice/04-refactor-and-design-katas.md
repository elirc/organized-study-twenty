# Refactor & Design Katas (6)

Paper/branch exercises — the deliverable is the design artifact plus (where marked) a spike diff. Self-grading criteria per kata.

## Kata 1: Split the base runner
The template-method base ([common-base-query-runner.service.ts#L60-L432](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-base-query-runner.service.ts#L60-L432)) owns throttling, permissions, complexity, hooks, context, enrichment. Design a decomposition (e.g. ordered pipeline stages as injectable classes) that preserves the can't-skip-security property.
Self-grade — Basic: responsibilities listed. Solid: stage interfaces sketched with ordering enforced by construction. Strong: you articulate why you might *not* ship it (blast radius vs pain today) — knowing when not to refactor is the senior skill.

## Kata 2: Design the outbox migration RFC
Write the actual RFC for [Project 1](03-senior-build-projects.md) using the template in [07-career-and-collaboration/02](../07-career-and-collaboration/02-writing-prs-and-rfcs.md).
Self-grade — Strong: includes a reconciliation metric defined precisely enough to alert on, and a phase-2 abort story.

## Kata 3: Exhaustiveness refactor of usePersistField
Refactor the guard chain ([usePersistField.ts#L97-L290](../../../packages/twenty-front/src/modules/object-record/record-field/ui/hooks/usePersistField.ts#L97-L290)) to a discriminated-union dispatch map keyed by `fieldDefinition.type` with a compile-time `never` check. Spike it.
Self-grade — Strong: your design keeps runtime *value* validation separate from compile-time *definition* dispatch, and you measured the diff in behavior for the morph-relation special case (L218-L264).

## Kata 4: Remove duplication across resolver factories
The 15+ factories in [workspace-resolver-builder/factories](../../../packages/twenty-server/src/engine/api/graphql/workspace-resolver-builder/factories) share the try/delegate/convert shape (see [create-one-resolver.factory.ts#L33-L63](../../../packages/twenty-server/src/engine/api/graphql/workspace-resolver-builder/factories/create-one-resolver.factory.ts#L33-L63)). Design the shared wrapper; identify which factories genuinely diverge (connection-shaped vs record-shaped responses).
Self-grade — Strong: your wrapper doesn't erase the useful type specificity of each factory's Args, and you can name the test that proves schema-shape neutrality (SDL snapshot).

## Kata 5: Boundary-leak hunt
Spend 90 minutes hunting one real boundary leak: feature code in `modules/` importing from `twenty-orm` internals, or front feature modules importing record-engine internals rather than public hooks. Write it up: import path, why it's a leak, the correct interface, migration cost.
Self-grade — Strong: you found one *defensible* leak (pragmatism beat purity for a reason) and said so, instead of flagging everything.

## Kata 6: Design a migration to per-delivery webhook jobs
Prerequisite thinking for M1/Project 2: today one job carries ≤20 deliveries ([call-webhook-jobs.job.ts#L66-L74](../../../packages/twenty-server/src/engine/metadata-modules/webhook/jobs/call-webhook-jobs.job.ts#L66-L74)). Design the move to one-job-per-delivery: queue volume math (×20), Redis memory, ordering implications, dedup, and a zero-downtime cutover (both job names consumable during deploy).
Self-grade — Strong: you handled the mixed-version window (old fan-out jobs in queue while new worker code deploys) explicitly — that window is where real incidents live.
