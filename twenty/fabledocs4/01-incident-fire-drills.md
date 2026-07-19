# 01 — Incident fire drills

Five simulated pages. Each gives you the alert and the evidence a real
on-call engineer would have — nothing more. Work each **CHECKPOINT** in
writing before scrolling past it. Suggested timebox: 45 minutes per drill for
the checkpoints; then read the debrief.

The scenarios are fictional; every mechanism they exercise is real and cited.

---

## Drill 1: "My new field doesn't exist"

**Page:** Support escalation. A workspace admin created a custom field
`dealStage` on their custom `Listing` object 30 minutes ago. The field shows
in Settings, but every record query using it returns
`Cannot query field "dealStage"`. Other workspaces are fine.

**Evidence available:**
- Settings UI shows the field (the metadata API returns it).
- GraphQL record queries reject the field name.
- Postgres: the workspace schema's `_listing` table **has** the column.
- No errors in server logs around field creation.

**CHECKPOINT 1:** Three layers could disagree here: metadata rows, the actual
Postgres table, and something else. What is the something else, and which two
layers do the symptoms prove are *in agreement*?

**CHECKPOINT 2:** Name the file you'd read first and the specific thing you'd
look for in it.

**Debrief:** Metadata and DDL agree (Settings reads metadata; the column
exists). The record GraphQL schema is **generated from a cached projection**
(`src/engine/api/graphql/workspace-schema.factory.ts` reading
`src/engine/workspace-cache/services/workspace-cache.service.ts`). The stale
layer is the workspace cache — the serving schema is a cache entry (ADR-006).
Investigate `getOrRecompute` keys and the workspace metadata version
(`src/engine/workspace-manager/workspace-version/`). Mitigation is cache
invalidation for that workspace; the real question for your write-up is *why*
invalidation didn't happen (in fabledocs2's telling: a dropped Redis pub/sub
message). Rule to keep: **"database right, cache wrong" is the default
hypothesis** for "my change didn't take effect."

---

## Drill 2: The silent webhook gap

**Page:** Customer email, four days after the fact: "Our ERP missed ~2,000
`company.updated` webhooks last Tuesday between 14:02 and 14:06 UTC. Your
status page shows nothing."

**Evidence available:**
- Customer's receiving endpoint was deploying at that time and returned 503s.
- Our audit log has `WEBHOOK_RESPONSE_EVENT` rows for that window:
  `success: false`, `status: 503`, one row per event.
- BullMQ dashboard: zero failed jobs, zero retries in that window.
- `call-webhook-jobs.job.ts` enqueues deliveries with `{ retryLimit: 3 }`.

**CHECKPOINT 1:** Zero failed jobs *and* thousands of failed deliveries. Those
two facts are only compatible under one condition. What is it?

**CHECKPOINT 2:** Read `src/engine/metadata-modules/webhook/jobs/call-webhook.job.ts`
lines 102–116. Now explain precisely why `retryLimit: 3` never fired.

**CHECKPOINT 3:** Your teammate proposes a one-line fix: rethrow in the catch.
The batch handler (`handle()`, lines 35–41) runs deliveries with
`Promise.all`. What new bug does the one-line fix create?

**CHECKPOINT 4:** Recovery: the customer needs their 2,000 events. Using only
what's already stored, sketch the replay script (data source, filter, loop).

**Debrief:** The catch block swallows every delivery failure, so BullMQ sees
every job succeed — retries only trigger on unhandled throws (ADR-010).
Rethrowing naively re-delivers the *entire 20-event chunk*, including the
ones that succeeded, to endpoints with no idempotency key to dedupe by. Real
fix is the capstone (doc 02). Recovery: query audit events for
`WEBHOOK_RESPONSE_EVENT` with `success: false` in the window, re-POST each
payload. The deeper lesson: failures were **recorded, not observed** — no
alert consumed the audit rows, so the customer found it before we did.

---

## Drill 3: The dashboard number is wrong

**Page:** "Our 'Active Opportunities' dashboard widget says 47. The
opportunities table view, same filter, shows 44 rows. Which is lying?"

**Evidence available:**
- Table view: 44 rows.
- Aggregate (COUNT) widget: 47.
- The filter involves a relation: opportunities whose *company* is tagged
  `enterprise`.
- Three opportunities were deleted (to trash, not purged) last week.

**CHECKPOINT 1:** Both numbers come from the same pipeline
(`graphql-query-runner/`, aggregations under `group-by/`). Given the
soft-delete design, state a specific hypothesis for how a COUNT and a row
list can diverge by exactly the number of trashed records.

**CHECKPOINT 2:** Which file family would you read to confirm — name the
query-builder classes involved in select vs. aggregate paths.

**CHECKPOINT 3:** Design a minimal reproduction as an integration test: what
records do you seed, what two queries do you run, what do you assert?

**Debrief:** The select builder applies `deletedAt IS NULL` to the root *and
joined relations* (`workspace-select-query-builder.ts`,
`workspace-soft-delete-query-builder.ts` — see fabledocs2 doc 03). The
standing bug class is aggregation paths applying the predicate to the root
but missing it (or applying it asymmetrically) on a *joined* relation — the
count includes rows whose join partner is trashed, the list doesn't, or vice
versa. Whatever you find, the repro test belongs in
`test/integration/` against real Postgres, because the bug lives in generated
SQL composition — exactly the class unit tests can't see (ADR-008).

---

## Drill 4: The half-applied migration

**Page:** A workspace admin reports: "I tried converting a text field to a
SELECT field. Got an error. Now I'm scared to touch it — is my data corrupt?"

**Evidence available:**
- Server logs: a `WorkspaceMigrationException` mid-way through the operation —
  action 4 of 7 failed (an index rebuild hit a lock timeout).
- Actions 1–3 applied: metadata row updated, new enum created.
- Actions 5–7 (column conversion, backfill, cache bump) never ran.

**CHECKPOINT 1:** In an imperative migration design, what would the admin's
next mutation on this object do? In *this* design
(`workspace-migration-builder/` diffing desired vs. **actual** state), what
happens instead on retry?

**CHECKPOINT 2:** What do you tell the admin, in one sentence, and what do you
check before telling them?

**CHECKPOINT 3:** The retry succeeds. Write the two-paragraph postmortem note:
what failed, why no data was at risk, and what (if anything) needs a code
change.

**Debrief:** This drill is ADR-004 paying rent. Because the builder diffs
against actual state, retry computes only the remaining delta (actions 4–7);
the applied actions 1–3 are recognized as already-converged and skipped.
Verify by inspecting the diff the builder produces (the actions are typed and
loggable) before running it. The imperative counterfactual — re-running
"create enum" against an enum that exists — would fail or fork state. The
code-change question from the postmortem: lock timeout on index rebuild is a
*scheduling* problem (run heavy DDL on the worker, not inline — the
architecture allows it but doesn't enforce it; fabledocs2 doc 01 flags this
exact erosion risk).

---

## Drill 5: The CORS red herring

**Page:** Frontend team: "Production is throwing CORS errors on every request
for some users! Did infra change the CORS config?"

**Evidence available:**
- Browser console: `blocked by CORS policy: No 'Access-Control-Allow-Origin'`.
- Only affects requests with a specific malformed JSON body produced by an
  old cached frontend bundle.
- Infra swears nothing changed. They're right.

**CHECKPOINT 1:** A CORS error that only occurs for *malformed requests* is a
signature. What is actually happening? (Hint: middleware runs in order, and
one of them parses JSON.)

**CHECKPOINT 2:** Find the file in this repo that exists specifically because
of this failure mode. What does it do, and what does its header comment say?

**Debrief:** An exception thrown in middleware that runs *before* the CORS
middleware produces an error response with no CORS headers; the browser then
reports a CORS failure instead of the real 4xx/5xx, sending everyone to
debug the wrong layer. `src/filters/unhandled-exception.filter.ts` (all 40
lines) exists to re-attach CORS headers to exception responses — its comment
is the two-sentence war story. Transferable lesson: **the browser reports the
last failure, not the first cause.** When "CORS errors" correlate with
request *content*, suspect an upstream exception, not CORS config.

---

## After the drills

For each drill, add one line to your lab notebook: the *generalized* rule
(not the specific bug). You should end with five rules resembling:

1. Database right, cache wrong — check the projection first.
2. Every catch in a job handler is a delivery-semantics decision.
3. Aggregates and lists diverge where join predicates diverge.
4. Declarative diff means retry-is-repair — but verify the diff before trusting it.
5. The browser reports the last failure, not the first cause.

If your five rules are close to these, move to the capstone.
