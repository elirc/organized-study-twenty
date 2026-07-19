# Behavioral STAR Stories — 8 Worksheets

Sourced from the tickets/projects in module 06 and from studying this repo. Fill in Result with *your* actual outcomes as you complete the work; numbers beat adjectives. Framing rule: you "contributed to / studied deeply" an open-source CRM — never imply employment. Rehearsal check for every story: under 2 minutes? concrete? ends with impact?

## Story 1: The at-most-once discovery
Prompts: "a time you found a subtle bug" / "disagreed about a technical decision" / "improved reliability."
Source: Flow 4 analysis + Ticket 1 + M1.
S: Open-source CRM; tenant webhooks are how customers' automations hear about data changes. T: I was studying delivery reliability. A: Traced delivery end to end; found the job's catch swallowed all errors ([call-webhook.job.ts#L102-L116](../../../packages/twenty-server/src/engine/metadata-modules/webhook/jobs/call-webhook.job.ts#L102-L116)) so the queue's `retryLimit: 3` never engaged — effectively single-attempt at-most-once. Rather than filing "bug," I pinned current behavior with unit tests and wrote up both interpretations (deliberate receiver-protection vs oversight) with a per-delivery retry design and its duplicate-delivery risk. R: [your outcome — test merged / discussion opened / design doc written].
Evidence to cite: the test, the anchors. Senior-signal: treated ambiguous intent as a decision to force, not a gotcha; designed the fix's failure mode (chunk re-delivery) before proposing it.
Resume bullet: Identified and test-pinned an at-most-once webhook delivery tradeoff in a 20-package open-source CRM; authored the retry/backoff design.

## Story 2: Learning a 1M+ line codebase fast
Prompts: "ramp-up on complex systems" / "learning something hard quickly."
Source: this curriculum's method (module 01).
S/T: Nx monorepo, metadata-generated GraphQL, custom ORM — no hand-written resolvers to find. A: systematic cartography: package graph → entry points → traced one write path end to end → built a glossary of the metadata vocabulary → verified by predicting behavior before reading. R: could locate the enforcement point for any permission question in minutes; produced flow docs others can onboard with.
Senior-signal: method over heroics; prediction-then-verification as the learning loop.
Resume bullet: Mapped and documented six end-to-end flows (auth, CRUD, async, metadata) of a production open-source CRM.

## Story 3: The race condition I closed (or designed the fix for)
Prompts: "hardest technical problem" / "concurrency bug."
Source: Ticket M6 (workflow start CAS).
S: Background workflow runs guarded by read-then-act status checks ([run-workflow.job.ts#L90-L95](../../../packages/twenty-server/src/modules/workflow/workflow-runner/jobs/run-workflow.job.ts#L90-L95)). T: queue redelivery + concurrent workers could double-start. A: replaced with a conditional UPDATE (compare-and-set), wrote a two-worker integration test that fails on the old code. R: [outcome].
Senior-signal: proved the race with a failing test *first*; knew why the status guard existed and preserved its intent.
Resume bullet: Eliminated a job double-execution race via compare-and-set state transitions with concurrency regression tests.

## Story 4: Measuring before optimizing
Prompts: "a performance problem" / "data-driven decision."
Source: M5 (bulk import) or the 04-performance drill.
A-core: back-of-envelope first (N creates × aggregate refetch + event fan-out math), then profiling to confirm the dominant term, then the batched fix, then the number after.
Senior-signal: you can quote the before/after numbers and the estimate that predicted them.

## Story 5: Saying no to my own refactor
Prompts: "a time you changed your mind" / "balancing tech debt vs delivery."
Source: Kata 1 (base-runner split).
A-core: designed the decomposition, then recommended *against* shipping it — blast radius across every API operation vs no current pain; wrote the watch-criteria that would revive it.
Senior-signal: engineering judgment is knowing when not to. Interviewers remember this story because almost nobody tells it.

## Story 6: Security finding handled responsibly
Prompts: "attention to detail" / "security awareness."
Source: review katas + bypass-flag audit drill.
A-core: audited every `shouldBypassPermissionChecks` call site; classified each as justified-system-context vs needs-review; proposed the justification-comment convention (Ticket-style small fix) rather than a scary rewrite.
Senior-signal: proportionate response; convention + review process over heroic patching.

## Story 7: Working a change through maintainers I don't control
Prompts: "influence without authority" / "conflict."
Source: any merged ticket + [07-career/03](../07-career-and-collaboration/03-maintainer-communication.md) templates.
A-core: evidence-first question ("is this intentional?"), small-slice proposal, concessions where the maintainer knew better. R: merged PR or a documented decision either way.

## Story 8: The migration with a rollback story
Prompts: "a risky change you de-risked."
Source: M8 (standard field + instance command) or Project 1 phases.
A-core: expand/contract sequencing, `up`+`down` both tested, feature-flag phases with per-phase abort criteria.
Senior-signal: "rollback is a deliverable" as your stated principle, with the tested `down` as proof.

---

Mapping table for quick recall under pressure:

| Prompt heard | Story |
| --- | --- |
| Conflict / disagreement | 1 or 7 |
| Hardest bug | 3 (or 1 told as debugging) |
| Ambiguity | 1 (ambiguous intent) or 2 |
| Mistake / changed mind | 5 |
| Tradeoff | 1, 4, or 8 |
| Leadership / initiative | 6 or 7 |
