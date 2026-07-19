# Command Cheatsheet

All commands **inferred** from [CLAUDE.md](../../../CLAUDE.md), [package.json](../../../package.json), and per-package `project.json` files unless marked verified. None were executed while authoring this suite (docs-only pass; no dependencies installed).

## Setup & run

| Task | Command | Status |
| --- | --- | --- |
| Install deps | `yarn install` | inferred (Yarn 4, Node ^24.5.0 required) |
| Dev services + env | `bash packages/twenty-utils/setup-dev-env.sh` (`--docker`, `--down`, `--reset`) | inferred |
| Start everything | `yarn start` (front + server + worker) | inferred |
| Front only / server only / worker | `npx nx start twenty-front` / `npx nx start twenty-server` / `npx nx run twenty-server:worker` | inferred |

## Quality gates

| Task | Command | Status |
| --- | --- | --- |
| Lint (fast, diff) | `npx nx lint:diff-with-main twenty-server` (`--configuration=fix`) | inferred |
| Typecheck | `npx nx typecheck twenty-front` / `twenty-server` | inferred |
| Format | `npx nx fmt twenty-front` | inferred |

## Tests

| Task | Command | Status |
| --- | --- | --- |
| Single test file (fastest loop) | `cd packages/twenty-server && npx jest "pattern"` | inferred |
| Package unit suites | `npx nx test twenty-server` / `twenty-front` | inferred |
| Server integration (real DB) | `npx nx run twenty-server:test:integration:with-db-reset` | inferred |
| Storybook build/tests | `npx nx storybook:build twenty-front` / `storybook:test twenty-front` | inferred |
| E2E | Playwright in `packages/twenty-e2e-testing` (see its `playwright.config.ts`) | inferred |

## Database & schema

| Task | Command | Status |
| --- | --- | --- |
| Reset DB | `npx nx database:reset twenty-server` | inferred |
| Init / migrate (prod-style) | `npx nx run twenty-server:database:init:prod` / `database:migrate:prod` | inferred |
| Generate instance command | `npx nx run twenty-server:database:migrate:generate --name <name> --type <fast\|slow>` | inferred |
| GraphQL codegen (front) | `npx nx run twenty-front:graphql:generate` (`--configuration=metadata` for /metadata) | inferred |

## Build & introspection

| Task | Command | Status |
| --- | --- | --- |
| Build (order matters) | `npx nx build twenty-shared` → `twenty-server` / `twenty-front` | inferred |
| Project graph | `npx nx graph` | inferred |
| Project targets | `npx nx show project twenty-server --json` | inferred |
| Affected-only | `npx nx affected -t test` | inferred |

Docker: compose files in [packages/twenty-docker](../../../packages/twenty-docker) (`docker-compose.dev.yml` for dev services). When a command here fails, trust the repo over this sheet: check the package's `project.json` targets first.
