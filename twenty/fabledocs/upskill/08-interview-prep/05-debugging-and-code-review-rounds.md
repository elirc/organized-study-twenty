# Timed Debugging & Code Review Rounds

Simulations of the "practical" interview hour. Run each with a timer; narrate aloud the whole time (interviewers grade the narration, silence reads as flailing). Grading rubrics at the bottom.

## Debugging round D1 (25 min): "Records vanish after refresh"
Source: [05-quality-engineering/03](../05-quality-engineering/03-systematic-debugging.md) Scenario 1.
Setup you state aloud: optimistic UI ([useCreateOneRecord.ts#L97-L188](../../../packages/twenty-front/src/modules/object-record/hooks/useCreateOneRecord.ts#L97-L188)) means the UI can show success the server never confirmed.
Interviewer follow-ups to rehearse: "The mutation returned 200 — now what?" (list query filters vs soft-delete vs row permissions); "How do you prevent the *class* of bug?" (surface mutation errors loudly despite optimism; e2e on create→refresh).

## Debugging round D2 (25 min): "Customer's webhooks stopped Tuesday"
Source: Scenario 2. The producer/router/consumer elimination, with the at-most-once punchline.
Follow-ups: "Audit shows failures with no status code — meaning?" (timeout or SSRF-block — the audit distinguishes blocked IPs, [call-webhook.job.ts#L103-L114](../../../packages/twenty-server/src/engine/metadata-modules/webhook/jobs/call-webhook.job.ts#L103-L114)); "Receiver was down for one hour — what did the customer lose?" (every event in that hour, permanently — no retries; segue to your fix proposal).

## Debugging round D3 (25 min): "Permission denied but the role looks right"
Source: Scenario 3. Query-shape bisection under deliberately opaque errors.
Follow-ups: "Why are the errors identical for different denial reasons?" (security vs debuggability trade — [STANDARD_ERROR_MESSAGE](../../../packages/twenty-server/src/engine/api/common/common-query-runners/errors/standard-error-message.constant.ts)); "Design better operator tooling without weakening the user-facing message" (server-side denial-reason logging keyed by request id).

## Debugging round D4 (20 min): "Workflow stuck in RUNNING"
Source: Scenario 5. Orphaned state machines and the janitor pattern ([workflow-handle-staled-runs.command.ts](../../../packages/twenty-server/src/modules/workflow/workflow-runner/workflow-run-queue/command/workflow-handle-staled-runs.command.ts)).
Follow-up: "How would you *prevent* orphans rather than sweep them?" (heartbeats/leases; CAS transitions — Ticket M6).

## Review round R1 (20 min): Kata 1 — inline webhook retry
From [04-code-reading-gym/04-review-katas.md](../04-code-reading-gym/04-review-katas.md). Deliver findings tiered (blocking/important/optional), each with consequence + alternative.
Follow-up: "Author pushes back: 'the queue retry re-sends all 20 deliveries.'" — concede the real problem, propose per-delivery jobs, note ordering/volume implications (Kata 6 of module 06).

## Review round R2 (20 min): Kata 2 — raw-SQL count endpoint
The security-shaped review. Must catch: identifier injection *and* the permission-bypass (raw query skips the enforcing builders — the bigger issue). Follow-up: "It's behind an admin flag, ship it?" (No — bypass of the permission layer is architectural, not audience-scoped; counts leak row existence across row-level predicates.)

## Review round R3 (20 min): Kata 4 — header-controlled permission bypass
The instant-block. Practice saying "this is a blocker" *kindly and unambiguously* in the same sentence, then routing to the legitimate alternative (system principal with a role).

## Rubrics

Debugging — Basic: reached root cause with hints. Solid: explicit layer-elimination narrated; checked evidence before theory. Strong: named the bug *class* and the regression test; used repo-specific instruments (audit log, queue state, metadata version) unprompted.

Review — Basic: caught the blocker. Solid: tiered findings, no style-blocking, consequences stated. Strong: every blocking finding paired with a viable alternative anchored to an existing pattern; tone you'd be glad to receive.

Do D2 + R1 back to back at least once: the same webhook knowledge powers both, which is exactly how a real onsite reuses your prep.
