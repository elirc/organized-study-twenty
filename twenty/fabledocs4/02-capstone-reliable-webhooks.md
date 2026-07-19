# 02 — Capstone: reliable webhook delivery (outbox + retries + idempotency)

This is the pack's centerpiece: you design and implement the real fix for
ADR-010. It's scoped like a mid-level engineer's quarter project, compressed.
Even if you only complete milestones 1–2, the design work alone will teach
you more distributed-systems engineering than a semester of reading.

**Ground rules:** work on a branch; never touch committed instance-command
`up`/`down` logic; every milestone ends with passing integration tests under
the **BullMQ driver**, not the sync driver (that constraint is half the
point — see ADR-008).

## The problem, restated precisely

Two independent gaps make webhook delivery at-most-once:

- **Gap A (loss before enqueue):** events are emitted in-process after the
  mutation commits (`src/engine/workspace-event-emitter/`); a crash between
  commit and enqueue loses the event with no trace.
- **Gap B (no retry after failure):** `call-webhook.job.ts` catches all
  delivery failures without rethrowing, so the fan-out's `{ retryLimit: 3 }`
  (`call-webhook-jobs.job.ts:72`) never fires.

Target semantics: **at-least-once delivery with consumer-side dedupability.**
(Exactly-once is not on the menu; know why before you start.)

## Milestone 0 — Design doc (do not skip)

Write 2–3 pages before any code. Must answer:

1. **Outbox schema.** The outbox row is written in the *same transaction* as
   the record mutation. Which schema does it live in — the workspace schema
   (per-tenant, ADR-003) or a shared core table? Argue both ways; commit to
   one. (Hint: the mutation's transaction is on the workspace datasource —
   that constrains you more than you think.)
2. **Relay design.** Poller vs. LISTEN/NOTIFY vs. hybrid. How does the relay
   mark rows dispatched without double-dispatching under two relay instances?
   (`FOR UPDATE SKIP LOCKED` is the classic answer — explain it in your own
   words, including *why* plain `FOR UPDATE` isn't enough.)
3. **Idempotency key.** What uniquely identifies a delivery —
   `(eventId, webhookId)`? Where does it go in the payload/headers, and what
   must the consumer do with it? Check what headers already exist
   (`X-Twenty-Webhook-Timestamp`, `-Signature`, `-Nonce` in
   `call-webhook.job.ts:60-70`) — is the nonce reusable as an idempotency key?
   (Careful: it's generated per *attempt*. Is that what you want? No. Say why.)
4. **Retry policy.** Backoff schedule, max attempts, and what happens after
   the last failure (dead-letter: stored where, alerting whom, replayable how?).
5. **Chunking.** Current design delivers 20 per job under `Promise.all`.
   Keep chunks (throughput) or switch to per-delivery jobs (clean retries)?
   If you keep chunks, how do you retry *only* the failed members?
6. **Back-compat.** Existing consumers (including
   `twenty-zapier/src/triggers/trigger_record.ts` — the comment in
   `call-webhook-jobs.job.ts` is your warning) must not break. New headers
   are additive; payload shape is frozen.
7. **Kill switch.** A feature flag (see `core-modules/feature-flag/`) gating
   the new path per workspace, with the old path as fallback.

**Review gate:** have someone senior (or your rubber duck, ruthlessly) attack
the design with: "relay crashes after POST but before marking dispatched —
what does the consumer see?" If your answer isn't "a duplicate, which is why
the idempotency key exists," return to step 3.

## Milestone 1 — Outbox write + relay skeleton

- Add the outbox entity and migration (follow the entity conventions;
  generate an instance command:
  `npx nx run twenty-server:database:migrate:generate --name webhook-outbox --type fast`).
- Write outbox rows inside the mutation transaction. Find the emission point
  by tracing where the ORM's execution path hands events to
  `workspace-event-emitter` — you want the transaction boundary, and finding
  it *is* the exercise.
- Relay job (cron-style; see `core-modules/cron/` for scheduling patterns):
  claim rows with `FOR UPDATE SKIP LOCKED`, enqueue fan-out, mark dispatched.

**Acceptance:** integration test (real Postgres + BullMQ) proving: (a) a
mutation whose transaction rolls back leaves *no* outbox row; (b) killing the
worker between commit and relay dispatch loses nothing — the row is picked up
on the next relay tick.

## Milestone 2 — Retry semantics

- Restructure delivery so a failed POST surfaces to BullMQ (rethrow or
  per-delivery jobs, per your Milestone 0 decision), with exponential backoff.
- Preserve the audit event per attempt (`WEBHOOK_RESPONSE_EVENT`) — now with
  an attempt number.

**Acceptance:** integration test with a stub HTTP endpoint that 503s twice
then 200s: exactly 3 attempts observed, one final success, audit rows for all
three, and — the ADR-010 regression test — successful chunk-mates delivered
**exactly once**.

## Milestone 3 — Idempotency + dead letter

- Stable idempotency key per (event, webhook) across attempts, sent as a
  header, documented in the OpenAPI webhook schema
  (`core-modules/open-api/utils/computeWebhooks.utils.ts`).
- After max attempts: dead-letter state on the outbox row, a metrics counter
  (`MetricsService` — and make it a *failure* counter this time; note the
  existing `JobWebhookCallCompleted` counts completions), and an admin-panel
  visible signal.

**Acceptance:** endpoint that always 500s → row reaches dead-letter after max
attempts; replaying the dead-letter re-sends with the *same* idempotency key.

## Milestone 4 (stretch) — Observability closure

Drill 2's lesson was "recorded, not observed." Wire a failure-rate signal:
counter of failed deliveries per workspace, exposed via
`core-modules/health/` or the admin panel, so four minutes of customer 503s
pages *us* first. Write the one-paragraph alert runbook.

## Self-assessment rubric

- **Junior-complete:** M0 doc coherent + M1 passing.
- **Mid-complete:** M2 with the exactly-once-chunk-mates test.
- **Senior-complete:** M3 + a crisp answer to "why is exactly-once impossible
  here, and what do we promise instead?"

Keep the design doc. Paired with fabledocs3's ADR-010 interview answer
("...and here's how I actually fixed it"), it converts a codebase-reading
exercise into a shippable-work story — the strongest artifact a junior
portfolio can hold.
