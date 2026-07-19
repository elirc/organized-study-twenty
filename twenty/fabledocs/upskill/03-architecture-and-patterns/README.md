# 03 — Architecture & Patterns

The judgment module. Everything here is about *why* the code is shaped this way, what it commits the team to, and how to recognize the same shapes elsewhere.

| File | Covers |
| --- | --- |
| [01-boundaries-and-layers.md](01-boundaries-and-layers.md) | The layer stack, what each owns, boundary wins and leaks |
| [02-data-model-and-persistence.md](02-data-model-and-persistence.md) | Multi-schema tenancy, dynamic DDL, migrations, transactions |
| [03-validation-auth-and-permissions.md](03-validation-auth-and-permissions.md) | Every validation layer; authn vs authz; the permission stack |
| [04-side-effects-async-and-reliability.md](04-side-effects-async-and-reliability.md) | Events, queues, retries, idempotency, delivery guarantees |
| [05-pattern-catalog.md](05-pattern-catalog.md) | 14 pattern cards with real anchors |
| [06-architecture-critique.md](06-architecture-critique.md) | Strengths, risks, prioritized improvements — doubles as system-design interview prep |

Vocabulary you'll use throughout (all interview words — definitions in context as they appear): **invariant** (a condition the system promises never to violate), **boundary** (a line where responsibility and data shape change), **contract** (a promise between components or to external consumers), **idempotency** (same operation twice = same result once), **blast radius** (how much breaks when this breaks), **consistency** (do all readers see the same truth), **observability** (can you tell what it did in production).
