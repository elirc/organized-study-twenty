# ADR-001: Nx monorepo with `twenty-shared` as the wire-type contract

## Status

Accepted — actively enforced by build ordering and import conventions.

## Context

Twenty ships a NestJS API, a large React SPA, a background worker, emails, a
Zapier integration, CLI/SDK packages, E2E tests, and docs — all of which must
agree on wire formats (event shapes, filter grammars, type guards) that change
frequently because the product's data model itself is user-defined (ADR-002).
Polyrepo coordination for that many synchronized surfaces means version-matrix
hell; a single unstructured repo means no dependency discipline.

## Decision

One Nx workspace (`nx.json`, `tsconfig.base.json`, Yarn 4 with
`yarn.config.cjs`) containing ~20 packages under `packages/`. Cross-cutting
types and utilities live in `packages/twenty-shared`, which must build before
anything that depends on it and is imported by both server and front via
subpath imports (`twenty-shared/utils`, `twenty-shared/database-events` — see
the imports at the top of
`packages/twenty-server/src/engine/metadata-modules/webhook/jobs/call-webhook-jobs.job.ts`).
Task orchestration (test, lint, typecheck, codegen) goes through Nx targets,
including diff-aware variants (`lint:diff-with-main`).

## Alternatives considered

- **Polyrepo with published npm packages** — rejected: every wire-type change
  becomes a publish/upgrade dance across 5+ repos; atomic cross-stack changes
  (schema + server + client + tests in one PR) become impossible.
- **Single app, no package boundaries** — rejected: the Zapier package, SDKs,
  and emails have genuinely different build targets and dependency sets;
  without boundaries the front would silently import server code.
- **Turborepo/pnpm** — plausible; Nx's project graph and per-target caching won,
  and the choice is low-stakes compared to the boundary discipline itself.

## Consequences

**Positive:** atomic cross-stack PRs; `twenty-shared` makes wire-contract drift
a compile error on both sides; Nx affected-graph keeps CI tolerable at this
repo size; one clone gives a contributor the whole product.

**Negative:** `twenty-shared` build-first ordering is a recurring new-developer
trap (stale `dist/` produces baffling type errors); Nx itself is a skill tax;
the monorepo makes it *possible* for any package to depend on any other, so the
boundary discipline is convention plus lint (`twenty-oxlint-rules` exists
partly for this) rather than physics.

## Evidence

`nx.json`; `tsconfig.base.json`; `packages/twenty-shared/` consumed by both
`twenty-server` and `twenty-front`; `packages/twenty-oxlint-rules/`;
CLAUDE.md's explicit "twenty-shared must be built first."
