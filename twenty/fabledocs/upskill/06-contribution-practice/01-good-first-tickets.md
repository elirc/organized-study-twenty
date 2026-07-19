# Good First Tickets (15)

Format per ticket: difficulty/time, skills, story, why it's a good contribution, anchors to read first, plan, risks, checks, and interview-story potential. The first two are written out in full; the rest are compressed but complete — expand them yourself as practice (that expansion *is* a skill).

---

## Ticket 1: Unit-test the webhook failure path
Difficulty: Easy — Estimated time: 3–5h
Skills practiced: Jest, NestJS DI mocking, async error semantics.
Story: As a maintainer, I want the webhook job's swallow-on-failure behavior pinned by a test so that future changes to retry semantics are deliberate.
Why this is a good contribution: zero product risk (test-only); documents currently-implicit behavior; directly useful to the retry-semantics discussion.
Acceptance criteria:
- [ ] Test: HTTP 500 → audit event `success:false`, handler resolves (no throw)
- [ ] Test: HTTP 200 → audit `success:true` + metric incremented
- [ ] Test: SSRF-blocked error → audit includes the private-IP error message
Read these anchors first: [call-webhook.job.ts#L43-L117](../../../packages/twenty-server/src/engine/metadata-modules/webhook/jobs/call-webhook.job.ts#L43-L117) — the unit under test; [transform-event-to-webhook-event.spec.ts](../../../packages/twenty-server/src/engine/metadata-modules/webhook/utils/__tests__/transform-event-to-webhook-event.spec.ts) — colocated test idiom.
Files likely touched: new `jobs/__tests__/call-webhook.job.spec.ts`.
Implementation plan: 1. Instantiate the job with mocked `AuditService`, `MetricsService`, `SecureHttpClientService` (return a fake axios). 2. Drive the three cases. 3. Assert audit payload shapes.
Illustrative fake-code shape (labeled):
```ts
// Illustrative fake code: not from this repo
const httpClient = { post: jest.fn().mockRejectedValue(Object.assign(new Error('boom'), { response: { status: 500 } })) };
```
What could go wrong: over-asserting exact payloads (brittle); missing the `void`-ed audit promise (assert via mock calls, not awaited results).
Suggested checks: `cd packages/twenty-server && npx jest call-webhook` (__inferred__).
Review questions: does this test freeze behavior the team may want to change? (Yes — that's its value; say so in the PR.)
Interview story potential: "I found an at-most-once delivery tradeoff and pinned it with tests to force an explicit decision."

## Ticket 2: Document the webhook wildcard contract
Difficulty: Easy — Estimated time: 2–4h
Skills practiced: reading routing code, contract documentation.
Story: As an integrator, I want the `object.operation` wildcard rules documented so I can subscribe correctly.
Why good: the four-pattern matching ([call-webhook-jobs.job.ts#L39-L46](../../../packages/twenty-server/src/engine/metadata-modules/webhook/jobs/call-webhook-jobs.job.ts#L39-L46)) is a public contract living only in code; the file itself warns about downstream dependents (L34-L37).
Acceptance criteria:
- [ ] Doc (or JSDoc-adjacent comment per repo comment style) enumerating `a.b`, `*.b`, `a.*`, `*.*` semantics with examples
- [ ] Cross-reference the OpenAPI webhook schema util ([computeWebhooks path noted in the code comment](../../../packages/twenty-server/src/engine/metadata-modules/webhook/jobs/call-webhook-jobs.job.ts#L34-L37))
Read first: the job file; [webhook.entity.ts](../../../packages/twenty-server/src/engine/metadata-modules/webhook/entities/webhook.entity.ts) (`operations` column).
What could go wrong: documenting aspiration instead of behavior — derive every claim from the match code.
Interview story potential: "I turned an implicit event-routing contract into documentation integrators can rely on."

---

## Ticket 3: Add a missing `down` assertion test for an instance command
Easy–Medium, 4–6h. Skills: migration thinking. Pick one recent command under `packages/twenty-server/src/database/commands`, write a test (or dry-run harness) asserting `down` reverses `up`. Read first: [CLAUDE.md](../../../CLAUDE.md) upgrade-commands section + `docs/UPGRADE_COMMANDS.md`. Risk: commands touching prod-only config. Interview story: "rollback as a tested deliverable."

## Ticket 4: Improve the unimplemented-field-type error in usePersistField
Easy, 2–3h. When the guard chain falls through, the throw at [usePersistField.ts#L282-L290](../../../packages/twenty-front/src/modules/object-record/record-field/ui/hooks/usePersistField.ts#L282-L290) stringifies the value but buries the field *name*. Add fieldName + type to the message (values may be sensitive — consider logging shape only, an intentionally debatable point to raise in the PR). Follows: CustomError conventions in twenty-shared. Interview story: "error messages are for the 2am debugger."

## Ticket 5: Storybook story for a record-field display component missing one
Easy, 3–5h. Find a display meta-type without stories (grep `record-field/ui/meta-types/display`); add stories per neighboring convention. Checks: `npx nx storybook:build twenty-front` (__inferred__). Interview story: "component contract made visible."

## Ticket 6: Add integration test: creating a record with an explicit invalid UUID
Easy, 3–4h. Pins [assert-is-valid-uuid at common-create-one-query-runner.service.ts#L100-L102](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-create-one-query-runner.service.ts#L100-L102) through the real API (error code, not 500). Suite home: [test/integration/graphql/suites](../../../packages/twenty-server/test/integration/graphql/suites). Interview story: "validation belongs to the contract, so I tested it at the contract."

## Ticket 7: Metric for webhook *failures*
Medium-Easy, 4–6h. Success increments `JobWebhookCallCompleted` ([call-webhook.job.ts#L98-L101](../../../packages/twenty-server/src/engine/metadata-modules/webhook/jobs/call-webhook.job.ts#L98-L101)); the catch path increments nothing. Add a failure counter key (follow `MetricsKeys` pattern). Small diff, real operability win ([05-quality-engineering/06](../05-quality-engineering/06-observability-and-operations.md) flagged this gap). Rejection risk: maybe metrics cardinality policy — ask first. Interview story: "made a silent failure mode measurable."

## Ticket 8: Glossary comment for the four-pattern event name computation
Easy, 1–2h. `computeEventName` ([workspace-event-emitter.ts#L61](../../../packages/twenty-server/src/engine/workspace-event-emitter/workspace-event-emitter.ts#L61)) + the split at [call-webhook-jobs.job.ts#L39](../../../packages/twenty-server/src/engine/metadata-modules/webhook/jobs/call-webhook-jobs.job.ts#L39) assume single-dot names — add a WHY comment per repo comment style stating the invariant. Interview story: "encoded an invariant where it's enforced."

## Ticket 9: E2E smoke for soft delete → restore
Medium-Easy, 5–8h. Playwright journey: create → delete → find in trash-ish state → restore. Read the e2e helpers in [twenty-e2e-testing/lib](../../../packages/twenty-e2e-testing/lib) first and reuse auth setup. Interview story: "tested the state machine users actually feel."

## Ticket 10: Type-narrowing cleanup in one file
Easy, 2–4h. Find a `as` cast in the runner pipeline replaceable with a guard from twenty-shared (`isDefined`). One file only — atomic diff. Rejection risk: some casts are load-bearing generic erasure ([03-type-system notes](../02-stack-and-language-mastery/03-type-system-and-contracts.md)); explain why yours isn't. Interview story: "shrank the unproven surface of a critical file."

## Ticket 11: Add missing self-check to a REST handler test gap
Medium-Easy, 4–8h. Pick one handler under [rest/core/handlers](../../../packages/twenty-server/src/engine/api/rest/core/handlers) without direct tests; add an integration case through the REST surface (parity with its GraphQL twin). Interview story: "two transports, one behavior — proved it."

## Ticket 12: Improve `.env.example` documentation for rate limiting
Easy, 1–2h. `API_RATE_LIMITING_*` knobs are read at [common-base-query-runner.service.ts#L356-L372](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-base-query-runner.service.ts#L356-L372); ensure the env example names them with units (ms) and defaults. Interview story: tiny, but demonstrates ops empathy.

## Ticket 13: Front — prevent double-submit on record creation
Medium, 6–10h. `loading` exists ([useCreateOneRecord.ts#L52](../../../packages/twenty-front/src/modules/object-record/hooks/useCreateOneRecord.ts#L52)) but callers may not disable actions; audit one creation surface (e.g. command menu "create company") and wire disabled-state. Verify the actual bug exists first (possible risk, not confirmed). Interview story: "idempotency starts in the UI."

## Ticket 14: Add a `retryLimit` constant with rationale
Easy, 1–2h. The magic `{ retryLimit: 3 }` at [call-webhook-jobs.job.ts#L72](../../../packages/twenty-server/src/engine/metadata-modules/webhook/jobs/call-webhook-jobs.job.ts#L72) → named constant + comment noting it applies to the *fan-out* job, not deliveries. Pure clarity diff that demonstrates you understood the semantics (most people won't). Interview story: seed for the retry-semantics conversation.

## Ticket 15: Contribute a debugging runbook
Easy, 3–5h. Turn [05-quality-engineering/03](../05-quality-engineering/03-systematic-debugging.md) Scenario 2 (webhooks stopped) into a short runbook doc where the team keeps ops docs. Interview story: "I write the doc I wished existed at 2am."

---

Working protocol for all tickets: branch → read anchors → write the test first where applicable → smallest diff → `lint:diff-with-main` + `typecheck` + targeted jest (__inferred__) → PR description per [07-career-and-collaboration/02](../07-career-and-collaboration/02-writing-prs-and-rfcs.md).
