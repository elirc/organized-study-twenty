# 05 — Flashcards: 50 cards for retention

Distilled from fabledocs 1–3. Use spaced repetition (import into Anki as
`Q ;; A`, or just cover the right column). Cards marked ★ are the ones I'd
expect anyone to answer cold after week 2; the rest are fair game by week 4.

## Architecture & metadata engine

1. ★ **What single product requirement drives this architecture?** — User-defined data model at runtime; objects/fields are metadata rows, not code (ADR-002).
2. ★ **Where do standard objects (company, person) get defined?** — As workspace entities in `src/modules/*` extending `base.workspace-entity.ts`, seeded into metadata — same rails as custom objects.
3. ★ **The three worlds a metadata mutation must change?** — Metadata rows, actual Postgres DDL, cached projections.
4. **What are the `flat-*` modules?** — Canonical flat projections of each metadata entity used by the diff/apply migration engine (~30 of them under `metadata-modules/`).
5. ★ **Migration engine in one phrase?** — Terraform-shaped: builder diffs desired flat state vs. actual state; runner applies typed actions (`workspace-migration-builder/` / `-runner/`).
6. ★ **Why is retry safe after a half-applied migration?** — The diff is computed against *actual* state, so retry applies only the remaining delta — retry-is-repair (ADR-004).
7. **Why was event-sourcing metadata rejected?** — Replay assumes reality matches the log; live DDL drifts; diff-vs-actual self-heals instead of confidently replaying wrongness.
8. ★ **Default hypothesis when "my change didn't take effect"?** — Database right, cache wrong: check `workspace-cache` before your code.
9. **What bounds cache staleness?** — Versioned cache keyed on workspace metadata version (`workspace-manager/workspace-version/`).
10. **The two server entrypoints?** — API (`src/main.ts`) and queue worker (`src/queue-worker/`), same codebase.
11. **Which package must build first and why?** — `twenty-shared`; both server and front import its wire types (ADR-001).
12. **Cost of monorepo boundaries being convention?** — Any package *can* import any other; discipline is lint (`twenty-oxlint-rules`) + review, not physics.

## Data access & tenancy

13. ★ **Tenancy model?** — Postgres schema per workspace, shared database; custom objects = real tables via runtime DDL (ADR-003).
14. ★ **Why can't shared tables + `workspaceId` column work here?** — Tenants have *different columns*; a shared table can't hold divergent shapes without regressing to JSONB/EAV.
15. ★ **Why is cross-tenant leakage structurally hard?** — A query targets one schema physically; leaking requires constructing the wrong schema name, not forgetting a WHERE.
16. **The tenancy scale ceiling?** — Table count = workspaces × objects; Postgres catalog bloat, autovacuum noise, slow dumps.
17. **The sharding escape hatch?** — `global-workspace-datasource/`: "which physical DB has this workspace" is a resolvable lookup, not a constant.
18. ★ **The one sanctioned entry point for record data access?** — `WorkspaceRepository` (`twenty-orm/repository/workspace.repository.ts`) and its query builders.
19. ★ **Where are permissions enforced, and why there?** — Inside the query builders (`permissions.utils.ts`) at construction time — four API surfaces inherit correct authz by construction (ADR-005).
20. **Why was Postgres RLS rejected for permissions?** — Policies would need generating/migrating per metadata change atop the DDL engine; debugging authz in-DB at that dynamism is worse.
21. ★ **Soft delete mechanics?** — `deletedAt` column; select builder adds `IS NULL` on the root *and every joined relation*; trash purge via `trash-cleanup/`.
22. **Classic soft-delete bug class?** — Aggregates vs. lists diverging when a join predicate is applied asymmetrically ("dashboard number is wrong").
23. **Rule when the safe path lacks a capability?** — Extend `WorkspaceRepository`; never route around it (the "ghost records" lesson).
24. **Who translates Postgres errors, and where?** — `twenty-orm/error-handling/`, inside the data layer, where metadata context can name the offending field.

## API design

25. ★ **Why is the record GraphQL schema generated at runtime?** — No compile-time entities exist (ADR-002); schema is built per workspace from cached metadata (`workspace-schema.factory.ts`).
26. ★ **The two GraphQL APIs and their construction models?** — Metadata API: code-first NestJS (compile-time entities). Record API: runtime-generated (ADR-006).
27. ★ **What is REST here?** — A facade: `api/rest/input-request-parsers/` translate onto the same `graphql-query-runner` pipeline. No second data path.
28. **Why one pipeline?** — Two data paths mean permissions/soft-delete implemented twice, guaranteed drift → security bugs.
29. **What else rides the pipeline?** — MCP (`api/mcp/`) and generated OpenAPI docs (`core-modules/open-api/`).
30. **Rejected "generic records API" — why?** — Pushes the type system onto clients: no introspection/codegen/validation for custom objects.
31. **Webhook payload versioning strategy?** — There isn't one; a code comment pointing at `twenty-zapier/src/triggers/trigger_record.ts` is the compatibility mechanism. Known debt.
32. **Cost of "API shape is a cache entry"?** — Schema rebuild latency on first request after metadata change; cache staleness = correctness bug.

## Error handling

33. ★ **The error-handling philosophy in one line?** — Domain throws coded exceptions; boundaries translate to protocol semantics (ADR-007).
34. **Why not NestJS HTTP exceptions in services?** — Same failure surfaces via GraphQL/REST/MCP/jobs; HTTP-shaped exceptions couple domain to one of four protocols.
35. **Why is `rest-api-exception.filter.ts` only ~31 lines?** — Errors arrive already shaped; the filter just maps codes to statuses. Thinness is the design working.
36. ★ **What is every `catch` in a job handler, really?** — A delivery-semantics decision: swallow = job "succeeded" = no retry.
37. **What does `unhandled-exception.filter.ts` exist for?** — Re-attaching CORS headers when pre-CORS middleware throws — browsers report the missing headers as a CORS error, masking the real failure.
38. **"CORS errors" correlated with request content means?** — An upstream exception before CORS middleware, not CORS config. Last failure ≠ first cause.

## Distributed systems & webhooks

39. ★ **Actual webhook delivery guarantee?** — At-most-once, twice over: no outbox (loss between commit and enqueue) + swallowed delivery failures (no retry) (ADR-010).
40. ★ **Why does `retryLimit: 3` never fire?** — BullMQ retries only on unhandled throw; `call-webhook.job.ts`'s catch swallows all failures, so every job "succeeds."
41. ★ **Why is naive rethrow the wrong fix?** — `handle()` runs a 20-delivery chunk under `Promise.all`; one rethrow re-delivers the whole chunk with no idempotency key to dedupe.
42. **The correct fix, in four parts?** — Transactional outbox + relay; per-delivery (not per-attempt) idempotency keys; surfaced failures with backoff; dead-letter + alerting.
43. **Why `FOR UPDATE SKIP LOCKED` in an outbox relay?** — Two relay instances can claim disjoint rows without blocking or double-dispatching.
44. **What security is already right in webhook delivery?** — HMAC-SHA256 signed payloads, timestamp/nonce headers, SSRF-guarded egress (`secure-http-client/`).
45. **"Recorded, not observed" means?** — Audit rows existed for every failed delivery; nothing consumed them, so the customer found the gap first. Signals need consumers.

## Testing & observability

46. ★ **Why real Postgres in integration tests?** — The risky code generates DDL/SQL; a mock faithful enough to test it *is* Postgres. Mocked-layer tests passed while the system broke.
47. ★ **What does `sync.driver.ts` erase?** — Retries, concurrency, redelivery, crash-loss. Any retry/idempotency "proof" under it is vacuous (ADR-008).
48. **Why codegen'd typed test clients?** — Runtime-generated schema (ADR-006) has few static checks; typed clients turn API drift into compile errors.
49. ★ **The four observability systems and their consumers?** — Logger (engineers reading), Sentry (engineers paged), metrics (dashboards/capacity), audit events (product feature + forensics).
50. **Why are audit/metrics calls `void`-ed, and the cost?** — Observability must never block or fail the operation observed; cost is silent loss — the forensic record is itself at-most-once.

## Self-test protocol

Week 2: all ★ cards, 90%+ cold. Week 4: full deck. For any miss, don't
re-read the card — open the cited file and re-derive the answer; the deck
indexes the code, it doesn't replace it.
