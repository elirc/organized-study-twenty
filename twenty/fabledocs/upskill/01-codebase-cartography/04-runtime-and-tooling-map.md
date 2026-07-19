# Runtime & Tooling Map

## Toolchain

| Concern | Tool | Where configured |
| --- | --- | --- |
| Package manager | Yarn 4 (Berry), workspaces | [package.json](../../../package.json#L23-L26), `yarn.config.cjs` |
| Task runner | Nx 22 | [nx.json](../../../nx.json), per-package `project.json` |
| Node version | `^24.5.0` (pinned, ES2023+ features available) | [package.json](../../../package.json#L18-L22) |
| Front build | Vite + Linaria (zero-runtime CSS-in-JS) | `packages/twenty-front/vite.config.ts` |
| Server build | Nx + tsc/webpack per project.json | `packages/twenty-server/project.json` |
| Unit tests | Jest per package (`jest.config.mjs`) | [jest.preset.js](../../../jest.preset.js) |
| Integration tests | Jest + real Postgres/Redis | [packages/twenty-server/test/integration/](../../../packages/twenty-server/test/integration) |
| E2E | Playwright | [packages/twenty-e2e-testing/playwright.config.ts](../../../packages/twenty-e2e-testing/playwright.config.ts) |
| Lint | oxlint + custom rules package | `packages/twenty-oxlint-rules/` |
| i18n | Lingui | `packages/twenty-front/src/locales` |
| CI | GitHub Actions, one workflow per package | [.github/workflows/ci-server.yaml](../../../.github/workflows/ci-server.yaml), [ci-front.yaml](../../../.github/workflows/ci-front.yaml) |

Everyday commands (all __inferred__ — see [09-reference/command-cheatsheet.md](../09-reference/command-cheatsheet.md) for the full set): `yarn start`, `npx nx typecheck twenty-server`, `npx nx lint:diff-with-main twenty-front`, `cd packages/twenty-server && npx jest <pattern>`.

## Runtime boundaries

Four distinct JS runtimes share this repo. Knowing which file runs where prevents an entire class of bugs:

1. **Browser** — `twenty-front`, `twenty-ui`. No Node APIs. State in Jotai + Apollo cache; styling compiled at build time by Linaria (so no runtime theme interpolation in template literals beyond what Linaria supports).
2. **Server (HTTP)** — `twenty-server` main process. NestJS DI; per-request auth context bound by [jwt-auth.guard.ts](../../../packages/twenty-server/src/engine/guards/jwt-auth.guard.ts#L23-L49); workspace context carried via AsyncLocalStorage-style storage ([orm-workspace-context.storage.ts](../../../packages/twenty-server/src/engine/twenty-orm/storage/orm-workspace-context.storage.ts), read via `getWorkspaceContext()` at [common-base-query-runner.service.ts](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-base-query-runner.service.ts#L316)).
3. **Worker** — same codebase, separate process (`nx run twenty-server:worker`), consumes BullMQ queues. Anything decorated `@Processor` runs here, *not* in the HTTP process.
4. **Shared** — `twenty-shared` must stay runtime-neutral: types, pure utils (`isDefined`), constants. If you import Node's `fs` here you break the browser build.

## Environment variables

Templates: `packages/twenty-server/.env.example`, `packages/twenty-front/.env.example` (no secrets committed). Server config is accessed through `TwentyConfigService` rather than raw `process.env` — see usage for rate limits at [common-base-query-runner.service.ts](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-base-query-runner.service.ts#L356-L372) and `NODE_PORT` at [main.ts](../../../packages/twenty-server/src/main.ts#L93). Transferable pattern: a typed config service centralizes parsing/defaults and makes config observable and testable; grepping for `twentyConfigService.get(` enumerates every knob.

## What CI actually runs (inferred from workflow files)

Per-package workflows (`ci-server.yaml`, `ci-front.yaml`, …) triggered on changed files (`changed-files.yaml`), plus a merge-queue workflow. Expect: lint, typecheck, unit tests, and for the server a job with Postgres/Redis services for integration tests. Before pushing, run the same trio locally: `lint:diff-with-main`, `typecheck`, targeted `jest`.

Drill: run `npx nx show project twenty-server --json` (__inferred__) and list which targets exist that CLAUDE.md never mentions. Self-grade — Basic: you list targets. Solid: you can say what 80% do. Strong: you identify which target CI runs that you weren't running locally, and add it to your pre-push habit.
