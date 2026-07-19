# 01 — Codebase Cartography

Before you can change a system safely you must know where things live, who owns what, and how the main flows run. This module builds that map for Twenty.

| File | What it gives you |
| --- | --- |
| [01-system-map.md](01-system-map.md) | Monorepo shape, runtime surfaces, ownership map, public vs private interfaces |
| [02-file-reading-order.md](02-file-reading-order.md) | 28 files in reading order with junior/mid/senior paths |
| [03-domain-glossary.md](03-domain-glossary.md) | Product and metadata vocabulary with code locations |
| [04-runtime-and-tooling-map.md](04-runtime-and-tooling-map.md) | Nx, Yarn 4, Vite, Jest, env vars, runtime boundaries |
| [05-key-flows.md](05-key-flows.md) | Six end-to-end flows traced with anchors, drills, and interview angles |

Transferable skill: **cartography is a repeatable procedure**, not repo trivia. Inventory the packages → find the entry points → find where contracts are defined → find where side effects happen → trace one write path end to end. Practice it here; reuse it at every new job. Interviewers ask "how do you ramp up on a large codebase?" — after this module you answer with a method, not vibes.
