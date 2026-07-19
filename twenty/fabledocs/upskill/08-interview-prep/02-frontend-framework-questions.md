# Frontend Framework Questions — 10 Cards

Same protocol: 90 seconds aloud first.

## Q1: How do you keep a table with thousands of cells responsive in React?
Testing: state architecture, not memo trivia.
Repo anchor: per-cell selector subscriptions — [usePersistField.ts#L187-L189](../../../packages/twenty-front/src/modules/object-record/record-field/ui/hooks/usePersistField.ts#L187-L189); Jotai atom families.
Junior: "use React.memo."
Mid adds: subscription granularity — each cell subscribes to `{recordId, fieldName}`, so one edit re-renders one cell; memo without narrow subscriptions doesn't help because the *store* notifies everyone.
Senior includes: virtualization for DOM count; reading-without-subscribing (`store.get`) for event handlers; measuring with the profiler before any of it.
Follow-up: why is context a poor fit here? (Any consumer re-renders on any change.)

## Q2: Where should state live? Walk through your decision.
Repo anchor: three tiers — Apollo cache / Jotai / local ([02-stack/02-framework-mental-models.md](../02-stack-and-language-mastery/02-framework-mental-models.md))
Mid adds: server data belongs in the server cache (normalized, shared); UI state in atoms scoped as narrowly as possible; ephemeral in useState. Name a real example of each from the repo.
Senior includes: the failure mode of duplicating server data into client state (two sources of truth diverge — the repo's record *store* + Apollo cache duality is managed deliberately with `upsertRecordsInStore` sync points, [useCreateOneRecord.ts#L50](../../../packages/twenty-front/src/modules/object-record/hooks/useCreateOneRecord.ts#L50)).

## Q3: Explain optimistic updates and their rollback.
Repo anchor: [useCreateOneRecord.ts#L87-L188](../../../packages/twenty-front/src/modules/object-record/hooks/useCreateOneRecord.ts#L87-L188)
Junior: "update UI before server responds."
Mid adds: client-minted UUID so optimistic and real record share identity; cache write → mutation → reconcile in `update` → delete-from-cache on error, then rethrow.
Senior includes: which fields you *can't* fabricate (server-computed); when optimistic is wrong (frequent validation rejections); duplicate-submit interactions.

## Q4: What are stale closures in hooks? How do you catch them?
Repo anchor: `useCallback` dep array [usePersistField.ts#L292-L299](../../../packages/twenty-front/src/modules/object-record/record-field/ui/hooks/usePersistField.ts#L292-L299)
Mid adds: a handler capturing an old value because deps were incomplete; lint rule `exhaustive-deps` as the guard; reading fresh values via store handles (the repo's `store.get` trick) as a legitimate escape.
Senior includes: unstable deps causing effect storms — the symmetric bug; deps as a *contract* about when identity must change.

## Q5: Controlled inputs at scale — how does this repo persist a cell edit?
Repo anchor: draft state in the field input → commit event → typed guard → `updateOneRecord` + store write ([usePersistField.ts#L184-L281](../../../packages/twenty-front/src/modules/object-record/record-field/ui/hooks/usePersistField.ts#L184-L281))
Mid adds: persist on commit (Enter/blur), not per keystroke; no-op suppression via deep-equality (L266-L268) preventing junk mutations *and* junk events downstream.
Senior includes: the blast radius of a UI write — every mutation fans out to webhooks/workflows; the UI's equality check is reliability engineering, not just perf.

## Q6: How does Apollo's normalized cache work, and when does it bite?
Repo anchor: optimistic-effect utils ([modules/apollo/optimistic-effect](../../../packages/twenty-front/src/modules/apollo/optimistic-effect)); typename computation ([getObjectTypename](../../../packages/twenty-front/src/modules/object-record/cache/utils/getObjectTypename.ts))
Mid adds: entities keyed by `__typename:id`; mutations returning entities auto-merge; *lists* don't auto-update — hence the manual optimistic effects for root queries.
Senior includes: cache shape as an implicit contract with every query in the app; why generic record engines need `recordGqlFields` bookkeeping.

## Q7: React Server Components / SSR — does this app need them?
Repo anchor: conceptual — no direct anchor; Vite SPA ([packages/twenty-front](../../../packages/twenty-front))
Mid adds: a CRM behind login is a poor SSR fit (no SEO, heavy interactivity, per-user data); SPA + aggressive caching is coherent.
Senior includes: cost framing — RSC pays off for content-heavy, cache-shared pages; being able to argue *against* the shiny thing with reasons is the senior signal.

## Q8: A page is slow. Walk me through finding out why.
Repo anchor: the domain table in [05-quality-engineering/04](../05-quality-engineering/04-performance-thinking.md)
Mid adds: profiler → is it render, network, or data volume; check re-render breadth (subscription bug) vs slow query (server) vs payload size (over-fetching depth-1 fields, [useCreateOneRecord.ts#L58-L64](../../../packages/twenty-front/src/modules/object-record/hooks/useCreateOneRecord.ts#L58-L64)).
Senior includes: fix + guardrail (e.g., complexity caps exist server-side — [common-base-query-runner.service.ts#L397-L431](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-base-query-runner.service.ts#L397-L431)).

## Q9: How do you test React components and hooks?
Repo anchor: testing-library conventions + Storybook tests ([CLAUDE.md](../../../CLAUDE.md) testing section); mock surface [__mocks__/useCreateOneRecord.ts](../../../packages/twenty-front/src/modules/object-record/hooks/__mocks__/useCreateOneRecord.ts)
Mid adds: query by role/text, user-event, behavior not implementation; hooks via renderHook with providers.
Senior includes: what *not* to test in components (server contract — integration's job); story-based visual contracts.

## Q10: CSS-in-JS — runtime vs zero-runtime?
Repo anchor: Linaria ([CLAUDE.md](../../../CLAUDE.md); patched toolchain in [package.json#L34-L35](../../../package.json#L34-L35))
Mid adds: styled-components computes styles at runtime (per-render cost, runtime dep); Linaria extracts static CSS at build (constraint: no arbitrary runtime interpolation).
Senior includes: the toolchain-maintenance cost (the repo literally patches its Linaria transform) — zero-runtime is a build-complexity trade, and someone pays it.
