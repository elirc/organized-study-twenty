# System Design From This Repo

The exercise: **"Design a CRM where customers can define their own objects and fields, with automations and integrations."** You will be asked something isomorphic to this (design Airtable/Notion/HubSpot-lite). Walk the steps below; at each step you know what a production system *actually chose* — cite it.

## Step 1 — Requirements (5 min)

Functional: CRUD on standard + custom objects; saved views (filter/sort); per-role permissions; webhooks + workflow automations; email/calendar sync. Non-functional: multi-tenant isolation (hard requirement), sub-second reads on 100k-record tenants, schema changes without deploys, integrations must survive receiver flakiness.

- Junior answer: jumps to tables. Mid: enumerates functional + non-functional and *prioritizes isolation and custom fields as the hard parts*. Senior: also states what's out of scope (analytics, offline) and why.

## Step 2 — API sketch (5 min)

Choose GraphQL for the product UI + REST for integrators, both over one execution pipeline. Real system: generated per-tenant GraphQL schema ([workspace-schema.factory.ts#L23-L68](../../../packages/twenty-server/src/engine/api/graphql/workspace-schema.factory.ts#L23-L68)), REST handlers delegating to the same runners ([rest/core/handlers](../../../packages/twenty-server/src/engine/api/rest/core/handlers)). Stronger/simpler alternative for a v1: REST only with a `fields` query param — say so, and say when you'd graduate.

- Mid: one pipeline, two transports, with the security rationale. Senior: schema *caching* keyed by metadata version, complexity limits as DoS defense ([common-base-query-runner.service.ts#L397-L431](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-base-query-runner.service.ts#L397-L431)).

## Step 3 — Data model (10 min)

The custom-fields decision (EAV / JSONB / dynamic DDL) — present the table from [03-architecture/02](../03-architecture-and-patterns/02-data-model-and-persistence.md), choose per constraints. Twenty chose dynamic DDL in schema-per-tenant; simpler alternative: JSONB + GIN indexes (defensible at small scale — say what forces the graduation: relational queries across custom fields, per-field indexes, type guarantees).
Metadata model: objects/fields/relations as rows; **metadata version** as the coherence key between DDL, caches, and clients (cite per-request version binding, [jwt-auth.guard.ts#L29-L33](../../../packages/twenty-server/src/engine/guards/jwt-auth.guard.ts#L29-L33)).

- Junior: draws company/person tables. Mid: draws the metadata layer and one custom-field storage choice with a tradeoff. Senior: names the failure mode of their choice (partial DDL/metadata failure → reconciliation job) unprompted.

## Step 4 — Permissions (5 min)

Roles → object CRUD grants → row-level predicates → field restrictions, enforced *in the data layer* so every transport inherits it ([workspace-select-query-builder.ts#L342-L390](../../../packages/twenty-server/src/engine/twenty-orm/repository/workspace-select-query-builder.ts#L342-L390)). Mention the join loophole and that the real system closes it ([permissions.utils.ts#L279-L360](../../../packages/twenty-server/src/engine/twenty-orm/repository/permissions.utils.ts#L279-L360)).

## Step 5 — Async & integrations (10 min)

Record events → queue → webhook fan-out with HMAC signing, SSRF-blocked egress, 5s timeouts, chunked jobs ([Flow 4](../01-codebase-cartography/05-key-flows.md)). Design decision to surface proactively: **delivery guarantees.** The real system is at-most-once single-attempt (swallowed errors); your design should *choose* — propose at-least-once with backoff + DLQ + receiver-side dedup via event id, and mention the outbox gap (events emitted in-process post-write; crash = loss) and its fix ([03-architecture/04](../03-architecture-and-patterns/04-side-effects-async-and-reliability.md)).

- Mid: queue + signing + retries. Senior: outbox, idempotency keys, circuit breaker per endpoint, and the *cost* argument for why v1 might skip them.

## Step 6 — Scaling concerns (5 min)

Read replicas for read-heavy list views (real: [L332-L334](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-base-query-runner.service.ts#L332-L334) — note the staleness tradeoff); Redis for metadata caches (version-keyed); worker pool sizing vs webhook timeout math ([05-quality-engineering/04](../05-quality-engineering/04-performance-thinking.md) capacity framing); tenant fan-out cost of migrations as the scaling wall of schema-per-tenant.

## Variation prompts (practice each for 10 min)

1. **"Now make a tenant with 10M records fast."** — per-tenant index strategy (index metadata exists per object), replica routing, cursor pagination discipline, view-specific projections.
2. **"Now add real-time collaboration."** — the repo has SSE/event-stream machinery ([engine/subscriptions](../../../packages/twenty-server/src/engine/subscriptions)); discuss event fan-out to connected clients vs polling, and why in-process emit is *fine* for ephemeral live updates even though it's insufficient for webhooks — same event, different delivery requirements. That contrast is a senior-grade insight.
3. **"Now guarantee webhook delivery."** — outbox + per-delivery jobs + DLQ + replay API (Projects 1–2 are your full worked answer).
4. **"Now let customers write code that runs on events."** — sandboxed logic functions (repo has [logic-function](../../../packages/twenty-server/src/engine/core-modules/logic-function) + code-interpreter modules); isolation, quotas, cold starts, secrets.
5. **"10x traffic tomorrow."** — stateless API scale-out (context is per-request), Redis/queue headroom, DB write ceiling per tenant schema, and *measurement first*: which limit hits first and how you'd know ([05-quality-engineering/06](../05-quality-engineering/06-observability-and-operations.md)).

## Delivery tips

Draw the layer stack from [03-architecture/01](../03-architecture-and-patterns/01-boundaries-and-layers.md) early; it structures everything after. Spend your minutes on steps 3 and 5 — that's where this design has real decisions. When stuck, narrate the tradeoff table; interviewers grade the *decision process* over the conclusion.
