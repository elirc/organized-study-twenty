# Framework Mental Models

## React: render vs commit, and where state lives

Mental model: a render is a pure function of props+state producing a description; commit applies diffs to the DOM; effects run after commit. Re-render is triggered by state changes, not data changes — so *where* state lives decides how much re-renders.

Twenty's state placement strategy (three tiers, each with a job):

1. **Server cache — Apollo.** Record data lives in the normalized Apollo cache; optimistic effects write into it directly ([useCreateOneRecord.ts#L97-L134](../../../packages/twenty-front/src/modules/object-record/hooks/useCreateOneRecord.ts#L97-L134)).
2. **Global client state — Jotai.** Atoms created through a local convention wrapper, e.g. [currentWorkspaceMemberState.ts](../../../packages/twenty-front/src/modules/auth/states/currentWorkspaceMemberState.ts) using `createAtomState`. Atom *families* keyed by record/field give surgical re-renders: a cell subscribes to `recordStoreFamilySelector({recordId, fieldName})` ([usePersistField.ts#L187-L189](../../../packages/twenty-front/src/modules/object-record/record-field/ui/hooks/usePersistField.ts#L187-L189)) so editing one cell doesn't re-render the table.
3. **Local state — useState.** Ephemeral flags like `loading` ([useCreateOneRecord.ts#L52](../../../packages/twenty-front/src/modules/object-record/hooks/useCreateOneRecord.ts#L52)).

Transferable interview line: "Put state at the narrowest scope that all its readers share; use selector-grade subscriptions (atom families / cache selectors) so a 10,000-cell table re-renders one cell."

Sharp edges here: hooks like `usePersistField` return `useCallback`-wrapped closures with dependency arrays ([L292-L299](../../../packages/twenty-front/src/modules/object-record/record-field/ui/hooks/usePersistField.ts#L292-L299)) — a missing dep = stale closure bug; an unstable dep = wasted re-renders. The repo also *reads without subscribing* via `store.get(...)` inside the callback (L187) — a deliberate escape from re-render coupling worth noticing.

## Apollo cache as a client-side database

The cache is normalized by `__typename` + id. Optimistic UI = write a fabricated record ([useCreateOneRecord.ts#L109-L113](../../../packages/twenty-front/src/modules/object-record/hooks/useCreateOneRecord.ts#L109-L113)), update every query that should include it (`triggerCreateRecordsOptimisticEffect`, L124-L134), then reconcile with the server response in the mutation's `update` callback (L146-L162), and delete on error (L169-L186). Failure modes: forgetting a root-query filter (record appears in lists it shouldn't), and drift between optimistic and server-computed fields. This machinery lives in [modules/apollo/optimistic-effect/](../../../packages/twenty-front/src/modules/apollo/optimistic-effect).

## NestJS: DI, guards, decorators

Mental model: NestJS is a DI container plus request pipeline (middleware → guards → interceptors → resolver/controller → filters). Everything is a class with declared dependencies; the container wires them.

- **DI in practice:** the base runner injects a dozen collaborators with `@Inject()` properties ([common-base-query-runner.service.ts#L64-L95](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-base-query-runner.service.ts#L64-L95)) — property injection because subclasses shouldn't repeat constructor boilerplate. Template-method pattern: subclasses implement `run/validate/computeArgs/processQueryResult` (L161-L182).
- **Guards compose:** authentication ([jwt-auth.guard.ts](../../../packages/twenty-server/src/engine/guards/jwt-auth.guard.ts)) then workspace presence ([workspace-auth.guard.ts](../../../packages/twenty-server/src/engine/guards/workspace-auth.guard.ts)) then capability guards (`settings-permission.guard.ts`) — each does one check.
- **Custom decorators as registration:** `@Processor(MessageQueue.webhookQueue)` + `@Process(name)` ([call-webhook.job.ts#L15-L34](../../../packages/twenty-server/src/engine/metadata-modules/webhook/jobs/call-webhook.job.ts#L15-L34)) register job handlers via metadata reflection — same idea as `@Controller`/`@Get`, applied to queues. Request-scoped providers exist too ([run-workflow.job.ts#L25](../../../packages/twenty-server/src/modules/workflow/workflow-runner/jobs/run-workflow.job.ts#L25) uses `Scope.REQUEST`) — know that scope changes instantiation cost.

Sharp edge: this repo *generates* its GraphQL layer instead of using Nest's code-first resolvers for workspace data — [workspace-schema.factory.ts](../../../packages/twenty-server/src/engine/api/graphql/workspace-schema.factory.ts#L48-L67) hands `makeExecutableSchema` a map of factory-built closures. Nest decorators still power the *metadata* API. If you look for `@Resolver()` classes for `company` you will not find them — that confusion costs juniors hours.

## Pitfall checklist

- [ ] Effect for something an event handler should do (repo convention: handlers over effects, [CLAUDE.md](../../../CLAUDE.md)).
- [ ] Broad Jotai atoms where families/selectors belong.
- [ ] Injecting a request-scoped provider into a singleton (Nest will widen the scope silently).
- [ ] Mutation without optimistic rollback path.

## Drills

1. Trace which components re-render when one cell's value is persisted (usePersistField L270-L281 sets store *and* fires mutation). Strong: you explain why both writes are needed.
2. Find one more `@Processor` class and describe how its jobs are registered without any central list.

## Interview angle

- "How do you keep a huge table fast in React?" → atom families + narrow subscriptions ([08-interview-prep/02](../08-interview-prep/02-frontend-framework-questions.md), Q1–Q4)
- "Optimistic updates: how, and how do you roll back?" → Q5
- "Explain DI and why it helps testing" → [08-interview-prep/01](../08-interview-prep/01-js-ts-node-deep-dive.md) Q11
- "Guards vs middleware vs interceptors in Nest" → [08-interview-prep/03](../08-interview-prep/03-api-and-data-modeling-questions.md) Q8
