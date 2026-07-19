# ADR-009: Driver abstraction for every infrastructure dependency

## Status

Accepted — applied uniformly enough to be recognizable as house style; the
tension it creates (cloud-scale observability wants richer backends than the
lowest-common-denominator interface exposes) is unresolved.

## Context

Twenty is simultaneously a **self-hosted open-source product** (someone's $10
VPS, minimal dependencies, console logs) and a **managed cloud** (Sentry,
BullMQ workers, ClickHouse analytics, S3 storage). Hard-wiring cloud
infrastructure into the code would make self-hosting punitive; targeting only
the minimal stack would cripple cloud operations.

## Decision

Every infrastructure concern is a NestJS dynamic module with a driver
interface chosen by configuration at boot, following a visible
`*.module-definition.ts` / `*.module-factory.ts` pattern:

- **Queue**: `message-queue/drivers/bullmq.driver.ts` (Redis-backed prod) vs
  `sync.driver.ts` (inline, dependency-free).
- **Exception reporting**: `exception-handler/drivers/sentry.driver.ts` vs
  `console.driver.ts`.
- **Logger**: `core-modules/logger/` behind `interfaces/logger.interface.ts`
  with its own module-factory.
- **File storage, cache, email**: `core-modules/file-storage/`,
  `cache-storage/`, `email/` follow the same shape.
- **Metrics**: counters over the already-required Redis
  (`core-modules/metrics/metrics-cache.service.ts`) rather than mandating a
  Prometheus/OTel stack.
- Configuration is centralized in `core-modules/twenty-config/`.

The same seam serves testing: the sync queue driver is both the self-host
fallback and the unit-test double (ADR-008).

## Alternatives considered

- **Hard dependencies on the cloud stack** — rejected: kills the open-source
  self-hosting story that drives adoption.
- **Separate cloud and OSS builds/forks** — rejected: two artifacts drift, and
  community fixes stop applying to cloud (the open-core trap).
- **Full OpenTelemetry as the one abstraction for observability** — heavier
  than a console-vs-Sentry switch; the chosen interfaces are narrower and
  owned in-repo.

## Consequences

**Positive:** one artifact serves laptop, VPS, and cloud; tests get honest
seams instead of monkey-patching; each dependency is swappable in isolation;
boot-time factory wiring makes the deployment matrix legible in code.

**Negative:** lowest-common-denominator interfaces — Redis-counter metrics
can't express histograms/percentiles, so latency truth lives in Sentry traces
and the two don't join; every driver pair is a behavioral-parity liability,
and the sync-vs-BullMQ semantic gap (no retries/redelivery inline) is the
worst instance — a *testing* blind spot (ADR-008) and a *reasoning* blind spot
(developers internalize sync semantics and write ADR-010's bugs); N drivers ×
M modules is real maintenance surface.

## Evidence

The `module-definition.ts` / `module-factory.ts` pairs across
`src/engine/core-modules/` (`exception-handler/`, `logger/`,
`message-queue/`); the two `drivers/` directories named above;
`metrics/metrics-cache.service.ts`; `twenty-config/`;
`packages/twenty-docker/` as the self-host deployment target the pattern
exists to serve.
