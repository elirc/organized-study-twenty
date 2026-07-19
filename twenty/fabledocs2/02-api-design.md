# 02 — API design: a GraphQL schema that doesn't exist until you ask for it

## What it is and why it exists

Because the data model is defined per workspace at runtime (see 01), we cannot
write resolvers for it. There is no `Company` class to decorate — a workspace
might have renamed Company, deleted it, or added forty fields to it. So the record
API is **generated**:

- `src/engine/api/graphql/workspace-schema.factory.ts` builds a workspace-specific
  GraphQL schema from cached metadata.
- `src/engine/api/graphql/workspace-schema-builder/` emits the types;
  `workspace-resolver-builder/` wires generic resolvers onto them.
- All resolvers funnel into one execution engine:
  `src/engine/api/graphql/graphql-query-runner/` (parsers in
  `graphql-query-parsers/`, `group-by/` for aggregations), which sits on
  twenty-orm (doc 03).

Around that core there are three more API surfaces, all thin translations onto the
same runner:

- **REST**: `src/engine/api/rest/` — `rest-api.service.ts` plus
  `input-request-parsers/` translate REST requests into the same query pipeline.
- **MCP**: `src/engine/api/mcp/` — model-context-protocol surface for AI agents.
- **Metadata GraphQL**: `metadata-graphql-api.module.ts` — a *conventional*
  code-first NestJS GraphQL API for manipulating the metadata itself. This one CAN
  be code-first because metadata entities are fixed at compile time.

Keep those two GraphQL APIs distinct in your head: the **metadata API** (static,
code-first, changes the model) and the **core/record API** (dynamic, generated,
changes the data). Different modules, different failure modes, different caches.

## The hard decisions

### 1. Generate the schema per workspace vs. one generic "records" API

The rejected alternative was a generic API: `records(objectName: "company",
filter: JSON)`. One schema, no generation machinery, massively simpler server.

We rejected it because it pushes the entire type system onto every client. No
codegen'd types for user objects, no GraphQL validation of filters, no
introspection-driven tooling — the frontend would be dynamically typed at the
wire boundary, and this frontend is 100% "no `any` allowed" TypeScript. Generating
a real schema means Apollo caching, introspection, and field-level validation all
work as if the user's custom objects were hand-written.

The cost, which still bothers me: **schema build latency and memory**. Every
active workspace needs a built schema object in memory; invalidation is coupled
to the workspace cache; and the first request after a metadata change pays the
rebuild. We amortize with caching keyed on workspace metadata version, but "the
API's shape is a cache entry" is a sentence that should make you slightly
nervous. It makes me slightly nervous, and I built it.

### 2. REST as a facade over the GraphQL pipeline vs. a real REST implementation

The REST API (`src/engine/api/rest/`) does not have its own data path. Parsers
turn querystring filters into the same internal query representation the GraphQL
runner uses. One pipeline, one permission model, one set of bugs.

Rejected: a separate REST implementation with its own repository calls. It would
have been faster per-request (no translation layer) and more idiomatically
REST-ish (real HTTP caching, ETags). But two data paths means every permission
check, soft-delete rule, and field-transformation exists twice, and they *will*
drift. I've lived that drift at a previous company; the security tickets write
themselves. Facade wins.

The wart: REST error semantics are translated GraphQL error semantics (see
`rest-api-exception.filter.ts`, all 31 lines of it), so status codes are
occasionally lawyerly — you'll see 400s where a REST purist expects 422. Known,
accepted, documented here so you don't "fix" it and break the SDK.

### 3. Webhook payloads are a public API we never versioned

Open `src/engine/metadata-modules/webhook/jobs/call-webhook-jobs.job.ts` and read
the comment at the top of `handle()`:

> If you change that function, double check it does not break Zapier trigger in
> `packages/twenty-zapier/src/triggers/trigger_record.ts`. Also change the openApi
> schema for webhooks in `.../open-api/utils/computeWebhooks.utils.ts`.

That comment *is* our versioning strategy, and I'm telling you plainly: it's a
confession. The webhook payload shape is consumed by every customer integration
plus our own Zapier package, and its compatibility contract is enforced by a
code comment and reviewer vigilance. The rejected alternative — explicit payload
schema versions with per-webhook version pinning — was cut for scope when
webhooks shipped, and retrofitting it now means a migration across every
customer's stored webhook config. It's the single piece of API-design debt I'd
pay down first with free time.

## Evidence to go read

- `src/engine/api/graphql/workspace-schema.factory.ts` — start here, follow the imports
- `src/engine/api/graphql/graphql-query-runner/graphql-query-parsers/` — how a GraphQL filter AST becomes an internal query
- `src/engine/api/rest/input-request-parsers/` — the same, from querystrings; note how little logic lives here
- `src/engine/api/graphql/workspace-query-runner/workspace-query-hook/` — pre/post hooks; this is where cross-cutting behavior (e.g., contact enrichment triggers) attaches to queries without touching the runner
- `src/engine/core-modules/open-api/` — the OpenAPI schema is *also* generated from metadata; REST docs are per-workspace too

## A production incident this design prevented

A customer's integration hammered the REST API with a filter on a field their
admin had just deleted. Because REST rides the generated pipeline, the request
failed **validation** against the current workspace schema — a clean 400 with the
field name — instead of reaching the database and producing a raw
`column does not exist` Postgres error (which, in the two-pipeline world, is
exactly what would have leaked: internal schema names, table naming conventions,
the works). Single-pipeline design meant the schema cache invalidation from doc
01's incident had *also* already been fixed for REST, for free, because there is
no "for REST" — there's one pipeline. That's the payoff of decision #2 and I'd
make it again without hesitation.

## Before you move on, you should be able to answer:

1. Why can the metadata GraphQL API be code-first NestJS while the record API
   must be generated? What compile-time fact differs between them?
2. Trace a REST `GET /rest/companies?filter=...` to the twenty-orm layer: which
   files touch the request, in order, and where would a permission denial occur?
3. If you need to add a field to the webhook payload, what three places (per the
   in-code comment) must you check, and why is that process itself a design smell?
