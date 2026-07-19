# 05 — Quality Engineering

How this repo stays correct, fast, secure, and observable — and how you demonstrate those instincts in interviews and PRs.

| File | Covers |
| --- | --- |
| [01-testing-strategy.md](01-testing-strategy.md) | The test layers, what belongs where, flake prevention |
| [02-writing-tests-here.md](02-writing-tests-here.md) | 6 concrete test recipes with commands |
| [03-systematic-debugging.md](03-systematic-debugging.md) | The method + 5 repo-specific scenarios |
| [04-performance-thinking.md](04-performance-thinking.md) | Hotspot map, measure-first discipline |
| [05-security-checklist.md](05-security-checklist.md) | Threats mapped to real defenses + pre-merge checklist |
| [06-observability-and-operations.md](06-observability-and-operations.md) | Logs, metrics, audit, health; "how would I know it broke?" |

The unifying idea: quality is **invariants plus evidence**. Tests pin invariants; observability produces evidence; debugging is the disciplined search when evidence contradicts invariants.
