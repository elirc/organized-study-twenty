# Interview translation

Each ADR compressed into a ~90-second spoken answer — first person, tradeoffs
up front, no jargon you can't defend — plus the follow-up an interviewer is
most likely to throw next. Practice these out loud; 90 seconds is roughly
200–230 words.

---

## ADR-001 — Monorepo with a shared wire-type package

**90-second answer:**
"The system had a server, a big React app, a worker, SDKs, a Zapier
integration — all sharing wire formats that changed constantly. We used an Nx
monorepo with one package, twenty-shared, holding the types that cross the
wire. The payoff is that a contract change is a single atomic PR: schema,
server, client, and tests move together, and drift becomes a compile error on
both sides instead of a runtime surprise between repo versions. The cost is
real: build ordering — shared must build first, and stale builds confuse new
engineers — plus the tooling tax of Nx itself. And a monorepo makes bad
dependencies *possible*; nothing physically stops the frontend importing
server code, so we backed the boundaries with lint rules. I'd still choose it:
for a product where one data-model change touches five surfaces, atomic
cross-stack changes are worth more than repo-level isolation. The alternative
— publishing versioned npm packages between polyrepos — turns every contract
change into a multi-repo release train, and teams start batching changes to
avoid the pain, which is how contracts rot."

**Likely follow-up:** "How did you keep CI times sane as the monorepo grew?"
(Talk about Nx affected-graph builds and diff-aware lint targets like
`lint:diff-with-main`.)

---

## ADR-002 — The data model is data

**90-second answer:**
"The core product requirement was that customers define their own objects and
fields at runtime — a real estate firm adds 'Listing' with a commission field
and it needs to behave exactly like the built-in objects. So the data model
isn't code; it's rows in a metadata store, and everything downstream — the
ORM, the GraphQL schema, REST, permissions, webhooks — is generated or
resolved from that metadata per request. Even the standard CRM objects are
defined through the same metadata path, so standard and custom objects ride
identical rails and features can't fork between them. The alternatives were a
JSONB blob or EAV tables, and both make custom fields second-class — you lose
typed columns, real indexes, honest foreign keys. The price we paid is that
nothing about the data layer is knowable at compile time, which forced us to
build a custom ORM, a runtime schema generator, and a migration engine —
three pieces of infrastructure most products get for free from their
framework. And the serving path depends on a cached projection of the
metadata, so cache staleness becomes a *correctness* bug, not a performance
bug. That's the deal: first-class customization in exchange for owning your
own platform layer."

**Likely follow-up:** "How do you keep the metadata cache consistent with the
actual database schema?" (Versioned cache keys per workspace metadata version;
stale reads become detectable rather than merely eventual.)

---

## ADR-003 — Schema-per-workspace tenancy

**90-second answer:**
"Because tenants have *different columns*, the usual tenancy answer — shared
tables with a workspaceId column — was structurally unavailable: a shared
company table can't hold one tenant's sixty custom fields and another's
three. Database-per-tenant gives great isolation but multiplies pools,
backups, and migrations by tenant count. We took the middle: one Postgres
database, one schema per workspace, and custom objects become real tables via
runtime DDL. Isolation becomes structural — a query physically targets one
schema, so cross-tenant leakage requires constructing the wrong schema name,
not just forgetting a WHERE clause. And custom data gets real indexes and
real query plans. The tradeoff I'm most honest about: table count grows as
tenants times objects, and Postgres catalogs aren't designed for hundreds of
thousands of tables — catalog bloat, autovacuum noise, slow dumps. There's a
scale ceiling. We hedged by centralizing datasource resolution so 'which
physical database holds this workspace' is a lookup, not a constant — meaning
we can shard workspaces across clusters later without touching query code.
Platform-wide upgrades also become O(tenants), so we built explicit
iterate-all-workspaces command machinery rather than pretending one migration
fits all."

**Likely follow-up:** "At what point does this break, and what's your sharding
story when it does?"

---

## ADR-004 — Declarative diff/apply metadata migrations

**90-second answer:**
"Adding a custom field has to change three worlds: metadata rows, actual
Postgres DDL, and caches — and runtime DDL can't always share a transaction
with the metadata write, so partial failure is a *normal* condition. The
imperative version — a service that writes rows, runs DDL, invalidates caches
inline — leaves metadata and the real schema silently disagreeing when step
three of seven fails. We rebuilt it Terraform-style: every metadata entity has
a flat canonical projection, a mutation produces desired state, a builder
diffs desired against *actual* state, and a runner applies typed actions.
The killer property is retry-is-repair: after a partial failure, rerunning
computes only the remaining delta, so convergence is a design property
instead of an ops runbook. We considered event-sourcing the metadata, but
replay assumes reality matches the log — and live databases drift — whereas
diffing against actual state self-heals. The cost is ceremony: one new
property touches the entity, its flat type, a comparator, a builder, and an
action handler. New engineers grumble for a month. I'd pay it again, because
the bug class it eliminates — schema drift you discover weeks later — is the
one that destroys customer trust."

**Likely follow-up:** "How do you handle a migration that fails halfway in a
way that *isn't* retryable — say, a data backfill that violates a new
constraint?"

---

## ADR-005 — Permissions and soft-delete inside the query builders

**90-second answer:**
"We had four API surfaces — GraphQL, REST, MCP for AI agents, and background
workflows — and every one must apply identical row and field permissions plus
soft-delete filtering. The conventional layering puts those checks in
services, which means four chances per feature to forget one, and a single
miss is a tenant-data leak. So we inverted it: we built a custom ORM layer
where the query builders themselves inject the permission predicates and the
deletedAt filtering — including on joins — at query-construction time. A
caller *can't* build an unauthorized query through the sanctioned path; every
surface inherits correct authorization by construction, which we proved when
the MCP surface arrived late and got authz for free. We considered Postgres
row-level security, but generating and migrating RLS policies on top of an
already-dynamic DDL engine, per metadata change, was worse than application
enforcement. The honest costs: we own a large bespoke ORM, TypeORM upgrades
are risky because we subclass its internals, and an escape hatch for raw
database access necessarily exists for system paths — so the final guarantee
is convention plus review, not physics. Our rule when the safe path lacks a
capability: extend the safe path, never route around it."

**Likely follow-up:** "What stops a developer from using the raw escape hatch,
concretely — and has anyone?"

---

## ADR-006 — Runtime-generated GraphQL, one pipeline, facades

**90-second answer:**
"Since the data model is runtime-defined, there's no Company class to write
resolvers for — so we generate the GraphQL schema per workspace from cached
metadata, and wire generic resolvers into a single execution engine. We
rejected the simpler generic API — records-of-objectName with JSON filters —
because it pushes the whole type system onto clients: no introspection of
custom objects, no codegen'd types for our strictly-typed frontend, no
server-side filter validation. Generating a real schema means custom objects
behave like hand-written API — Apollo caching, validation, tooling all work.
Then the key discipline: REST, MCP, and the OpenAPI docs are *facades* that
translate onto that same pipeline. No second data path, ever — because two
data paths means permissions and soft-delete implemented twice, and they will
drift; that drift is how security tickets are born. The costs: the API's
shape is effectively a cache entry, so schema-cache staleness is a
correctness bug and rebuild latency hits the first request after a metadata
change; and REST semantics are occasionally translated-GraphQL-flavored. The
piece I'd flag as debt: webhook payloads became a public contract with no
version field — compatibility is enforced by a code comment pointing at the
Zapier package. That's a confession, not a strategy."

**Likely follow-up:** "How do you version an API whose schema differs per
tenant and changes at runtime?"

---

## ADR-007 — Coded exceptions, boundary translation

**90-second answer:**
"The same domain failure surfaces through GraphQL, REST, MCP, and job logs —
and GraphQL doesn't even have HTTP statuses in-band. So the philosophy is:
domain code throws typed exceptions with machine-readable codes and no
protocol opinion; the boundaries own translation. The REST exception filter
is about thirty lines because by the time an error reaches it, it's already
well-shaped — that thinness is the design working. Below that, the data layer
translates raw Postgres errors into domain exceptions before they escape, so
constraint names and schema internals never reach a client, and the
translation happens where metadata context exists to say *which field's*
constraint failed. We rejected idiomatic NestJS HTTP exceptions from services
— convenient, but it couples domain logic to one of four protocols. Costs:
per-module exception classes are ceremony, and legacy corners still throw
bare Error, so there's a catch-all filter for unknown shapes. The subtlest
consequence: background jobs have no protocol boundary, so a catch block in a
job isn't error *handling* — it's a delivery-semantics decision. Swallow an
exception and you've silently chosen at-most-once. We learned that one the
hard way, which is its own story."

**Likely follow-up:** "Tell me the hard-way story." (Segue directly into
ADR-010.)

---

## ADR-008 — Real Postgres in integration tests

**90-second answer:**
"Our riskiest code generates DDL and dynamic SQL, and a mock faithful enough
to test that *is* Postgres, minus fidelity. Early mocked-data-layer tests
passed while the system broke — worse than no tests, because they bought
false confidence. So the strategy is a deliberate pyramid: fast unit tests
for logic, a serious integration tier that runs against real seeded Postgres
with a database reset, and thin end-to-end. Two supporting decisions matter.
First, integration tests call the API through *generated typed clients*, so
when the runtime-generated schema changes shape, tests break at compile time
instead of returning mysterious nulls — with a runtime-generated API, that's
one of our few static contract checks. Second, the job queue is behind a
driver interface, and tests run a synchronous inline driver — fast and
deterministic, but it erases retries, concurrency, and redelivery. I'm very
explicit with the team that any test 'proving' retry or idempotency behavior
under the sync driver proves nothing; queue-semantics correctness needs the
real driver. The remaining debt I'll name unprompted: reset is per-run, not
per-test, so tests can couple through leftover data — and since workspaces
are cheap in our architecture, per-test workspace isolation is the obvious
unbuilt fix."

**Likely follow-up:** "Give me a concrete bug the integration tier caught that
units couldn't." (Cross-schema foreign-key name resolution in generated DDL —
correct parts, wrong composition, only visible against real Postgres.)

---

## ADR-009 — Driver abstractions for infrastructure

**90-second answer:**
"The product is self-hosted open source *and* a managed cloud — same code
serving someone's ten-dollar VPS and our fleet. So every infrastructure
concern sits behind a driver interface chosen by config at boot: the queue is
BullMQ in production or a synchronous inline driver with zero dependencies;
exception reporting is Sentry or console; storage, cache, email follow the
same pattern. Metrics deliberately ride Redis — which we already require —
rather than mandating a Prometheus stack a self-hoster would have to run. The
alternatives were hard cloud dependencies, which kill the self-hosting story
that drives adoption, or separate OSS and cloud builds, which drift into the
open-core trap. The same seams double as test seams, so tests swap drivers
instead of monkey-patching. Two honest costs. Interfaces sink to the lowest
common denominator: Redis counters can't do histograms, so latency truth
lives in Sentry traces and the two datasets don't join. And every driver pair
is a behavioral-parity liability — the worst being sync-versus-BullMQ, where
the inline driver has no retries or redelivery, which quietly trains
developers on the wrong mental model of the queue. Abstraction gave us reach;
the parity gaps are the interest payments."

**Likely follow-up:** "How do you test that two drivers actually behave the
same?" (Contract-test suites run against both drivers — and admitting where
that's aspirational is a strong answer.)

---

## ADR-010 — Event emission and webhook delivery semantics

**90-second answer:**
"This is the decision we made by accident, and it's my favorite to talk about
because the code contains the fossil record. Webhook delivery: mutations emit
events in-process after commit — no transactional outbox — a fan-out job
matches subscriptions and enqueues delivery jobs *with a retry limit of
three*, and the delivery job wraps the HTTP POST in a catch block that logs
an audit event and doesn't rethrow. Walk the chain: the queue only retries on
an unhandled throw; the catch swallows every failure; therefore the retry
limit can never fire — it's dead configuration. Delivery is at-most-once
twice over: a crash between commit and enqueue loses the event, and a
customer's 503 gets exactly one attempt. Someone intended retries and someone
else's catch block silently vetoed them, and no document recorded either
choice. The fix isn't just deleting the catch — the batch runs under
Promise.all, so a naive rethrow re-delivers the whole chunk to endpoints that
already succeeded. Correct is an outbox written in the mutation's
transaction, per-delivery idempotency keys, rethrow, and dead-letter
alerting. My takeaway: in async code, every catch block is a
delivery-semantics decision, and unwritten decisions still get made — that's
why I write ADRs now."

**Likely follow-up:** "Design the fix: walk me through the outbox schema, the
relay, and how consumers deduplicate."

---

## Using these in a real interview

Don't recite. Pick the two ADRs closest to the job's domain (data-heavy role:
003/004/005; platform role: 001/009; reliability role: 008/010) and lead with
those. Every answer above follows the same skeleton — *constraint, decision,
rejected alternative, cost I'll admit unprompted* — and that skeleton, more
than any content, is what reads as senior.
