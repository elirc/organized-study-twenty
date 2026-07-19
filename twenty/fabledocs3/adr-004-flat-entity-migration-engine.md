# ADR-004: Declarative flat-entity diff/apply for metadata changes

## Status

Accepted — the ~30 `flat-*` modules show a completed, systematic migration to
this pattern; it is the codebase's most distinctive internal architecture.

## Context

A metadata mutation ("add field") must change three worlds atomically-ish:
metadata rows, actual Postgres DDL (ADR-003), and cached projections
(ADR-002). Runtime DDL cannot always share a transaction with metadata writes,
so partial failure is a *normal* operating condition, not an edge case.
Imperative mutation services — write rows, run DDL, invalidate caches, inline —
make partial failure unrecoverable: metadata and the real schema silently
disagree, and every subsequent operation builds on the lie.

## Decision

Metadata changes are **declarative and diff-driven**, Terraform-style. Every
metadata entity has a "flat" canonical projection
(`src/engine/metadata-modules/flat-object-metadata/`, `flat-field-metadata/`,
`flat-view/`, `flat-webhook/`, `flat-role/`, …). A mutation produces a desired
flat state; `workspace-migration-builder/` (per-entity builders under
`.../workspace-migration-builder/builders/` — `field/`, `index/`, `webhook/`,
etc.) diffs desired against actual and emits typed migration actions;
`workspace-migration-runner/action-handlers/` executes them. A shared
`universal-flat-entity/` layer generalizes the comparison machinery.

## Alternatives considered

- **Imperative mutation services** — the apparent starting point this pattern
  replaced; rejected because partial failure produces silent drift with no
  convergence mechanism.
- **Event-sourced metadata** (append-only change log, state by replay) —
  rejected: replay assumes the target's real state matches the log's
  assumption, but runtime DDL against a live database drifts (failed steps,
  manual intervention); replay then produces confident wrongness, whereas
  diffing against *actual* state self-heals.
- **Two-phase commit across metadata + DDL** — not honestly available;
  arbitrary DDL and data backfills can't be transactionally coupled at size.

## Consequences

**Positive:** retry-is-repair — after partial failure, re-running computes only
the remaining delta, so convergence is a design property rather than an ops
runbook; the diff is inspectable before apply; every metadata entity gets
migration support by implementing the same builder/handler contract, which is
why views, webhooks, and roles all migrate uniformly.

**Negative:** heavy ceremony — one new property on one metadata entity touches
the entity, its flat type, comparator, builder, and action handler
(boilerplate across five directories); the flat projection is one more
representation to keep in sync mentally; error reporting UX is awkward because
"failed, please retry" is correct behavior but reads as flakiness.

## Evidence

Count the `flat-*` directories in `src/engine/metadata-modules/` (~30);
`src/engine/workspace-manager/workspace-migration/workspace-migration-builder/builders/`;
`.../workspace-migration-runner/action-handlers/`;
`.../universal-flat-entity/`; `workspace-migration.exception.ts` for the
failure taxonomy.
