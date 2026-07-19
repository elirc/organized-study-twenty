# 01 — Architecture: the metadata engine and the shape of everything else

## What it is and why it exists

Twenty is a CRM where customers define their own data model. A workspace admin can
create a "Property Listing" object with a "Commission Rate" currency field at 2pm
on a Tuesday, and by 2:00:05 that object has a Postgres table, a GraphQL API with
filtering and aggregation, REST endpoints, permissions, and UI views. That single
product requirement — **user-defined schema at runtime** — is the gravitational
center of this codebase. Almost every architectural oddity you'll find is a
consequence of it.

Concretely, the server is two applications wearing one NestJS trench coat:

1. **The core app** — auth, billing, workspaces, users, files, feature flags.
   Ordinary NestJS + TypeORM code against fixed tables. Lives in
   `packages/twenty-server/src/engine/core-modules/` (look at `auth/`, `billing/`,
   `workspace/`, `feature-flag/`).
2. **The metadata engine** — the machinery that stores *descriptions* of objects
   and fields (`src/engine/metadata-modules/`, see `object-metadata/`-style modules
   like `field-metadata/` and the whole family of `flat-*` modules), turns those
   descriptions into real DDL (`src/engine/workspace-manager/workspace-migration/`),
   and serves records through a runtime-built API (`src/engine/api/` and
   `src/engine/twenty-orm/`).

The monorepo around it is an Nx workspace (`nx.json`, `tsconfig.base.json`) with
~20 packages. The ones you'll live in: `twenty-server`, `twenty-front` (React 18 +
Jotai + Linaria), `twenty-shared` (types shared across the wire — build this first),
and `twenty-ui`.

## The hard decisions

### 1. Dynamic DDL per workspace vs. EAV vs. JSONB

This was the foundational fight. Three ways to let users define fields at runtime:

- **EAV** (entity-attribute-value rows): flexible, and your queries become
  unreadable five-way self-joins with no useful indexes. Every mature EAV system
  I've seen ends up rebuilding a query planner in application code. Rejected.
- **JSONB blob per record**: great write ergonomics, but you lose typed columns,
  cheap per-field indexes, and honest foreign keys — and the CRM's whole value is
  relational queries ("all opportunities on companies owned by X").
  Rejected, though it was close.
- **Real DDL** — each workspace gets its own Postgres schema, each user-defined
  object gets a real table, each field a real column. This is what we do. The
  `workspace-migration-builder/` diffs the desired metadata state against current
  state and emits actions; `workspace-migration-runner/` executes them (see the
  per-entity builders in
  `src/engine/workspace-manager/workspace-migration/workspace-migration-builder/builders/`
  — there's one for `field`, `index`, `webhook`, `view`, and a dozen more).

What still bothers me: we bought query performance and integrity at the price of
**operational blast radius**. A tenant with 400 custom objects has 400 real tables.
Upgrades that touch every workspace (the `@RegisteredWorkspaceCommand` machinery)
are O(workspaces × objects) DDL statements. `pg_dump` on a busy instance is an
adventure. If I were starting today I'd still choose DDL, but I'd cap per-workspace
object count from day one instead of retrofitting limits.

### 2. The `flat-*` representation: diffing metadata instead of imperatively mutating it

Early on, metadata mutations were imperative: "create field" ran a service that
wrote metadata rows AND issued DDL AND invalidated caches, in one tangle. Bugs came
from partial application — metadata said one thing, the actual table said another.

The redesign: every metadata entity has a **flat** projection —
`flat-object-metadata/`, `flat-field-metadata/`, `flat-view/`, `flat-webhook/`,
etc. (count them in `src/engine/metadata-modules/`; there are ~30). A mutation
builds the desired flat state, the migration builder computes a diff, and the
runner applies it. It's Terraform-shaped: declarative core, imperative edges.

Rejected alternative: event-sourcing the metadata (append-only log of metadata
changes, rebuild state by replay). Beautiful on paper; we rejected it because
schema state must be *exactly* consistent with Postgres's actual catalog, and
replaying a log against a database whose real state drifted (manual hotfix,
failed migration) gives you confident wrongness. The diff approach self-heals
by construction.

The lingering tradeoff: the flat layer is a lot of ceremony. Adding one property
to one metadata entity touches the entity, the flat type, the comparator, the
builder, and the runner action handler. New engineers hate it for a month. It's
the tax we pay for "metadata and DDL can never disagree," and I'd pay it again,
but I won't pretend the boilerplate doesn't sting.

### 3. One NestJS process vs. splitting the metadata engine into its own service

We debated extracting the metadata engine into a separate service with its own
deploy cadence. Rejected: the engine and the record API share the workspace cache
(`src/engine/workspace-cache/`), the ORM (`src/engine/twenty-orm/`), and the
datasource pool (`src/engine/workspace-datasource/`). Splitting them means either
duplicating the cache (two sources of truth for "what does this workspace's schema
look like") or putting a network hop inside every query. We kept one process and
split the *worker* instead — background jobs run in a separate entrypoint
(`src/queue-worker/`) against the same code.

What bothers me: the API process still does DDL. A giant metadata sync can eat
connections and starve request traffic. The mitigation is queueing heavy work to
the worker, but the boundary is discipline, not architecture — nothing *stops* a
new resolver from running expensive DDL inline. That's exactly the kind of rule
that erodes.

## Evidence to go read

- `packages/twenty-server/src/engine/workspace-manager/workspace-migration/` — builder vs. runner split, `universal-flat-entity/`
- `packages/twenty-server/src/engine/metadata-modules/flat-field-metadata/` — pick one flat module and trace it end to end
- `packages/twenty-server/src/engine/workspace-cache/services/workspace-cache.service.ts` — `getOrRecompute` is called from everywhere; understand its keys
- `packages/twenty-server/src/main.ts` and `src/queue-worker/` — the two entrypoints
- `CLAUDE.md` at the repo root — the working agreements, kept deliberately short

## A production incident this design caused

**The stale-schema Tuesday.** A workspace admin added a field; the migration ran
fine; the GraphQL API kept rejecting the field for ~40 minutes. Root cause: the
workspace cache held the old flat metadata, and the invalidation message was
dropped when a Redis failover blipped mid-publish. Nothing was *wrong* in Postgres
— metadata and DDL agreed perfectly (the flat-diff design did its job) — but the
serving layer was reading a cached projection of reality with no TTL bound tight
enough to save us.

The fix was making cache entries versioned against a workspace metadata version
(see `src/engine/workspace-manager/workspace-version/`) so a stale read is
detectable, not just eventual. The lesson that generalizes: **in this codebase,
"the database is right but the cache is wrong" is the default hypothesis** for any
"my change didn't take effect" report. Start there. It will save you days.

## Before you move on, you should be able to answer:

1. When an admin creates a custom field, name every layer that changes state —
   in order — from the GraphQL mutation to the moment a record query can filter
   on that field. (There are at least five.)
2. Why did we choose real DDL per workspace over JSONB, and what specific query
   capability would we have lost with JSONB?
3. What guarantees does the flat-entity diff/apply design give us that an
   imperative "create field" service couldn't — and what class of bug does it
   *not* protect against (hint: re-read the incident)?
