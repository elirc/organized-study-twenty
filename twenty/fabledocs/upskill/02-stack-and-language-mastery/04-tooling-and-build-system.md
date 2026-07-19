# Tooling & Build System

## Why a monorepo, and what Nx buys

One repo holds the server, front, shared types, UI kit, SDK, and e2e suite ([package.json](../../../package.json#L46-L60)). The payoff: a cross-layer change (schema + API + UI + tests) is **one atomic PR** — no version dance between repos. The cost: build orchestration, which is Nx's job.

Nx mental model: a **project graph** (who imports whom) plus **task pipelines** ([nx.json](../../../nx.json)) declaring that, e.g., building `twenty-front` requires building its dependencies first — which is why CLAUDE.md says build `twenty-shared` first. Nx caches task outputs keyed by input hashes: unchanged package → cache hit → seconds instead of minutes. `lint:diff-with-main` targets exist precisely to keep feedback loops short on a huge codebase.

Commands that expose the machinery (all __inferred__):

```bash
npx nx graph                     # visualize the project graph
npx nx show project twenty-server --json
npx nx affected -t test          # only test what your change touches
```

Transferable: every serious monorepo tool (Nx, Turborepo, Bazel) is the same three ideas — graph, hashed inputs, cached outputs. Learn it once here and you can say so in interviews.

## Yarn 4 workspaces

Yarn Berry with `packageManager: yarn@4.13.0` pinning ([package.json](../../../package.json#L23)). Notable: the `resolutions` block (L27-L39) force-pins transitive deps (graphql, typescript) — one version across all workspaces prevents the classic "two graphql instances" runtime bug. There's even a patched dependency via `patch:` protocol (L34-L35) — vendoring a fix without forking. Engines block (L18-L22) hard-pins Node `^24.5.0` and rejects npm (`"npm": "please-use-yarn"` — a cute guardrail worth stealing).

## Vite + Linaria on the front

Vite dev server = native ESM, per-module transforms, instant HMR. Linaria compiles CSS-in-JS to static CSS **at build time** — styled-components ergonomics, zero runtime style recalculation. Constraint that bites: Linaria template literals are evaluated at build time, so they can't close over arbitrary runtime values the way styled-components can — dynamic styling goes through props/CSS variables. The custom Vite/wyw-in-js patch in resolutions (L34-L35) shows the team maintaining this toolchain edge actively.

## Module systems

Server code imports via absolute `src/...` paths (see any server file's imports, e.g. [common-base-query-runner.service.ts#L5-L57](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-base-query-runner.service.ts#L5-L57)); front uses `@/` aliases (e.g. [useCreateOneRecord.ts#L4-L33](../../../packages/twenty-front/src/modules/object-record/hooks/useCreateOneRecord.ts#L4-L33)); cross-package imports use package names (`twenty-shared/utils`). Aliases are declared in [tsconfig.base.json](../../../tsconfig.base.json). Sharp edge: `graphql-upload/graphqlUploadExpress.mjs` in [main.ts#L9](../../../packages/twenty-server/src/main.ts#L9) — an explicit `.mjs` import inside a CJS-built server is the ESM/CJS interop seam; expect friction whenever a dependency goes ESM-only.

## Where builds fail in practice

1. Forgot to build `twenty-shared` after changing it → stale d.ts, phantom type errors.
2. Added a Node-only import to `twenty-shared` → front build breaks.
3. Changed GraphQL operations without regenerating types → front typecheck fails (`graphql:generate` first).
4. Import cycle across packages → Nx graph error; fix the dependency direction, don't suppress.

## Drill

Run `npx nx graph` (__inferred__) and answer: which packages depend on `twenty-shared`? Which depend on `twenty-ui`? What would a circular dep between `twenty-server` and `twenty-front` even mean? Self-grade — Strong: you can explain why shared *types* flow through a package instead of `import type` from each other directly.

## Interview angle

- "Monorepo vs polyrepo — tradeoffs?" → atomic cross-layer changes vs build complexity; anchor to this repo's workspace list. ([08-interview-prep/01](../08-interview-prep/01-js-ts-node-deep-dive.md) Q13)
- "How do you keep CI fast on a large repo?" → task graph + caching + affected-only runs.
- "CJS vs ESM — what actually breaks?" → the `.mjs` import seam above. (Q12)
