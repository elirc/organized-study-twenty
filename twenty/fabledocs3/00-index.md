# Reverse-Engineered Architecture Decision Records — Twenty CRM

No ADRs were checked into this repository, but the code is full of decisions —
some deliberate, some accidents that hardened into load-bearing walls. This pack
reconstructs the 10 most significant ones from code evidence alone: file layout,
class design, naming patterns, and the places where a comment betrays what the
authors were worried about.

Each ADR follows the standard format — Context, Decision, Alternatives
Considered, Consequences, Status — and cites the specific files that constitute
the evidence. "Status" reflects my read of the code today: a decision the code
still enforces is **Accepted**; one the code half-honors is flagged.

| ADR | Decision | Area |
| --- | --- | --- |
| [001](adr-001-nx-monorepo-shared-types.md) | Nx monorepo with `twenty-shared` as the wire-type contract | Layering / structure |
| [002](adr-002-metadata-driven-data-model.md) | The data model is data: user-defined schema at runtime | Foundational |
| [003](adr-003-schema-per-workspace-tenancy.md) | Postgres schema-per-workspace tenancy with real dynamic DDL | Data access |
| [004](adr-004-flat-entity-migration-engine.md) | Declarative flat-entity diff/apply for metadata changes | Data access / consistency |
| [005](adr-005-twenty-orm-authz-in-data-layer.md) | Custom ORM layer with permissions and soft-delete in query builders | Data access / security |
| [006](adr-006-generated-graphql-single-pipeline.md) | Runtime-generated GraphQL schema; REST/MCP/OpenAPI as facades over one pipeline | API contract |
| [007](adr-007-coded-exceptions-boundary-translation.md) | Coded domain exceptions, translated only at API boundaries | Error handling |
| [008](adr-008-real-postgres-integration-tests.md) | Integration tests against real Postgres; swappable sync queue driver | Testing |
| [009](adr-009-driver-abstraction-self-hosting.md) | Driver abstraction for every infrastructure dependency | Structure / deployability |
| [010](adr-010-event-emission-webhook-delivery.md) | In-process event emission, queue fan-out, no outbox — at-most-once webhooks | Distributed systems |

Then: [Interview translation](99-interview-translation.md) — each ADR as a
90-second spoken answer, with the follow-up question an interviewer would
most likely ask next.

**Reading order note:** ADR-002 is the root of the dependency tree. 003, 004,
005, and 006 are all forced moves once 002 is accepted; reading them in
sequence shows how one product requirement propagates through an entire
architecture.
