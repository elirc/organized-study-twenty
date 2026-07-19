# Fast Track — One Weekend in Twenty

Goal: by Sunday night you can run the app, trace two end-to-end flows, make one safe change, run one test, and explain a flow out loud as if an interviewer asked.

## Saturday morning — run it

All commands are __inferred__ from [CLAUDE.md](../../CLAUDE.md), [package.json](../../package.json#L40-L45), and `packages/twenty-utils/setup-dev-env.sh` (not executed while writing these docs). Requires Node `^24.5.0` and Yarn `>=4.0.2` ([package.json](../../package.json#L18-L22)).

```bash
yarn install                                   # inferred
bash packages/twenty-utils/setup-dev-env.sh    # inferred — starts Postgres + Redis, creates DBs, copies .env files
yarn start                                     # inferred — front (3001) + server (3000) + worker, see package.json "start"
```

If setup fails, read the script before retrying — it is idempotent and supports `--docker`, `--down`, `--reset`. Env templates live at `packages/twenty-server/.env.example` and `packages/twenty-front/.env.example`. For UI login, click "Continue with Email" and use the prefilled dev credentials (per [CLAUDE.md](../../CLAUDE.md)).

## Saturday afternoon — the first 10 files, in order

1. [CLAUDE.md](../../CLAUDE.md) — commands, conventions, architecture summary. The repo's own onboarding.
2. [package.json](../../package.json) — workspaces list; notice `start` runs server + front + worker concurrently. Three runtimes, one command.
3. [packages/twenty-server/src/main.ts](../../packages/twenty-server/src/main.ts#L27-L96) — NestJS bootstrap: session middleware, body-size limits, GraphQL upload middleware on `/graphql` and `/metadata`. Two GraphQL endpoints is a load-bearing fact.
4. [workspace-schema.factory.ts](../../packages/twenty-server/src/engine/api/graphql/workspace-schema.factory.ts#L23-L68) — the heart: per-workspace GraphQL schema built from metadata SDL + generated resolvers.
5. [create-one-resolver.factory.ts](../../packages/twenty-server/src/engine/api/graphql/workspace-resolver-builder/factories/create-one-resolver.factory.ts#L28-L64) — how one generated resolver delegates to a query-runner service.
6. [common-base-query-runner.service.ts](../../packages/twenty-server/src/engine/api/common/common-query-runners/common-base-query-runner.service.ts#L101-L159) — the template-method pipeline every read/write goes through: throttle → validate → permissions → parse → hooks → execute.
7. [workspace-select-query-builder.ts](../../packages/twenty-server/src/engine/twenty-orm/repository/workspace-select-query-builder.ts#L342-L352) — permissions enforced *inside the ORM*, not just in resolvers.
8. [useCreateOneRecord.ts](../../packages/twenty-front/src/modules/object-record/hooks/useCreateOneRecord.ts#L84-L188) — optimistic create with client-generated UUID and rollback on error.
9. [usePersistField.ts](../../packages/twenty-front/src/modules/object-record/record-field/ui/hooks/usePersistField.ts#L61-L302) — how an inline cell edit becomes a mutation.
10. [call-webhook.job.ts](../../packages/twenty-server/src/engine/metadata-modules/webhook/jobs/call-webhook.job.ts#L43-L117) — a background job with HMAC signing, SSRF protection, and a deliberate swallow-all-errors catch.

## Saturday evening — trace two flows

Trace these with the files above open (full templates in [01-codebase-cartography/05-key-flows.md](01-codebase-cartography/05-key-flows.md)):

1. **Create a company (UI → DB):** button → `useCreateOneRecord` (optimistic cache write, UUID minted client-side) → `createOneCompany` mutation → generated resolver → `CommonCreateOneQueryRunnerService` → base pipeline → `WorkspaceInsertQueryBuilder.execute()` → Postgres → `CREATED` event emitted ([workspace-insert-query-builder.ts](../../packages/twenty-server/src/engine/twenty-orm/repository/workspace-insert-query-builder.ts#L222-L275)).
2. **That event becomes a webhook:** event batch → `CallWebhookJobsJob` matches subscriptions with wildcards and enqueues chunks ([call-webhook-jobs.job.ts](../../packages/twenty-server/src/engine/metadata-modules/webhook/jobs/call-webhook-jobs.job.ts#L39-L74)) → `CallWebhookJob` signs and POSTs with a 5s timeout.

Pause-and-predict before reading: *what happens to the optimistic record if the mutation fails?* Then check [useCreateOneRecord.ts](../../packages/twenty-front/src/modules/object-record/hooks/useCreateOneRecord.ts#L164-L188).

## Sunday — one change, one test, one teach-back

- **Safe change:** pick any UI copy string or add a log line in a resolver factory; run `npx nx typecheck twenty-front` or `twenty-server` (__inferred__). Revert after.
- **One test:** `cd packages/twenty-server && npx jest merge-field-values` (__inferred__) — runs [merge-field-values.util.spec.ts](../../packages/twenty-server/src/engine/api/graphql/graphql-query-runner/utils/__tests__/merge-field-values.util.spec.ts), a pure-util unit test that needs no DB.
- **Teach-back (interview rehearsal):** explain flow 1 aloud in under 3 minutes. Must include: where the record ID comes from, where authorization happens, what "optimistic" means, and one failure mode. Record yourself. If you said "the frontend creates the ID so the create is idempotent and the UI can render instantly," you're already answering at mid-level.

## What the fast path skips

Metadata sync and migrations (the hardest subsystem), workflow engine internals, email/calendar sync, billing, the Nx build graph, and all of testing strategy. Those live in modules 01–05. Don't skip the [security checklist](05-quality-engineering/05-security-checklist.md) before your first real PR.
