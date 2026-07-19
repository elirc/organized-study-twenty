# Tier 1 Junior Missions

### Mission 1: The App's Heartbeat
**Tier:** Junior
**Time Estimate:** 45 minutes
**Goal:** Explain frontend and backend bootstrapping and where control first transfers.

**The Concept:**
Bootstrapping is the handoff from framework runtime to app-specific behavior. In this CRM, it is like opening the office and assigning desks before customer work can begin.

**Design Intent Before You Read the Code:**
1. `packages/twenty-front/src/index.tsx` mounts UI root.
2. `packages/twenty-front/src/modules/app/components/App.tsx` composes foundational providers.
3. `packages/twenty-server/src/main.ts` configures HTTP runtime and middleware.
4. `packages/twenty-server/src/app.module.ts` wires feature modules and route middleware.

**Find It In The Code:**
Open:
- `packages/twenty-front/src/index.tsx:1-14`
- `packages/twenty-front/src/modules/app/components/App.tsx:1-46`
- `packages/twenty-server/src/main.ts:27-96`
- `packages/twenty-server/src/app.module.ts:49-146`

Inline annotations:
```tsx
const root = ReactDOM.createRoot(...); // create browser render root
root.render(<App />); // transfer control to app provider stack
```
```ts
const app = await NestFactory.create(AppModule, ...); // instantiate backend runtime
app.use(session(...)); // attach session middleware
await app.listen(...); // start network listener
```

**The Aha Moment:**
Boot code is not setup noise; it is where cross-cutting guarantees are established.

**Socratic Checkpoint:**
1. What breaks if `App` is rendered without provider wrappers?
2. Why does backend call `useContainer` before handling requests?
3. Where is workspace auth context attached for REST paths?
4. Why does frontend mount once but route many times?
5. Which bootstrap line most affects security posture?

**How to self-grade:**
Strong answers cite exact lines and one concrete runtime consequence.

**Connects To:**
Mission 2, Mission 8.

### Mission 2: The Folder Mental Map
**Tier:** Junior
**Time Estimate:** 35 minutes
**Goal:** Build a practical map of where frontend/backend responsibilities live.

**The Concept:**
Folder structure is operational ownership.

**Design Intent Before You Read the Code:**
Use `packages/twenty-front/src`, `packages/twenty-server/src`, `packages/twenty-shared/src` as primary domains.

**Find It In The Code:**
Open:
- `packages/twenty-front/src/index.tsx:1-14`
- `packages/twenty-front/src/modules/app/components/AppRouterProviders.tsx:1-96`
- `packages/twenty-server/src/app.module.ts:49-146`

**The Aha Moment:**
Correct folder intuition cuts debugging time more than syntax skill.

**Socratic Checkpoint:**
1. Which folder owns transport concerns?
2. Which folder owns workspace guards?
3. Why keep shared enums outside frontend/backend?
4. Which folder likely contains domain redirects?
5. Where would you add a new guarded metadata route?

**How to self-grade:**
Strong answers map responsibility to concrete file anchors.

**Connects To:**
Mission 5, Mission 12.

### Mission 3: TypeScript Is a Contract
**Tier:** Junior
**Time Estimate:** 30 minutes
**Goal:** Explain how a typed contract prevents runtime ambiguity.

**The Concept:**
Types are executable team agreements.

**Design Intent Before You Read the Code:**
Inspect user and enum contracts.

**Find It In The Code:**
Open:
- `packages/twenty-front/src/modules/auth/states/currentUserState.ts:4-20`
- `packages/twenty-shared/src/types/ViewSortDirection.ts:1-4`
- `packages/twenty-server/src/engine/metadata-modules/view-sort/entities/view-sort.entity.ts:44-50`

**The Aha Moment:**
A good type limits invalid states before production sees them.

**Socratic Checkpoint:**
1. What field in `CurrentUser` is critical for admin gating?
2. Why enum for sort direction?
3. What runtime bug appears if direction is free text?
4. Where is type cast risk visible?
5. What type boundary crosses frontend/backend here?

**How to self-grade:**
Mention one prevented bug and one remaining risk.

**Connects To:**
Mission 17.

### Mission 4: Your First React Component
**Tier:** Junior
**Time Estimate:** 20 minutes
**Goal:** Understand props and conditional render in a small component.

**The Concept:**
Components are deterministic view functions plus props contracts.

**Design Intent Before You Read the Code:**
Use a simple auth title component.

**Find It In The Code:**
Open `packages/twenty-front/src/modules/auth/components/Title.tsx:6-35`.

**The Aha Moment:**
Even tiny components encode design-system invariants.

**Socratic Checkpoint:**
1. Why use `Pick<TitleProps, 'noMarginTop'>` in styled block?
2. What happens when `animate=true`?
3. Why default `noMarginTop=false`?
4. What if `children` were optional and absent?
5. What part is logic vs style contract?

**How to self-grade:**
Explain each prop and one failure mode.

**Connects To:**
Mission 6.

### Mission 5: Your First Node.js Route
**Tier:** Junior
**Time Estimate:** 30 minutes
**Goal:** Trace one guarded REST route from request to service call.

**The Concept:**
Routes are narrow adapters; guards and services do the heavy lifting.

**Design Intent Before You Read the Code:**
Use dashboard duplication route for compact signal.

**Find It In The Code:**
Open `packages/twenty-server/src/modules/dashboard/controllers/dashboard.controller.ts:11-24`.

**The Aha Moment:**
Most route safety lives in decorators and middleware, not handler body size.

**Socratic Checkpoint:**
1. Why three guards?
2. Where does workspace identity come from?
3. Why DTO return type?
4. What breaks without exception filter?
5. How would frontend call this endpoint?

**How to self-grade:**
Map request -> guard -> context -> service -> response.

**Connects To:**
Mission 13, Mission 14.

### Mission 6: Props Are a Typed Contract
**Tier:** Junior
**Time Estimate:** 25 minutes
**Goal:** Trace typed data from parent decision to child behavior.

**The Concept:**
Typed props are interface boundaries between UI responsibilities.

**Design Intent Before You Read the Code:**
Use settings flag wiring through router.

**Find It In The Code:**
Open:
- `packages/twenty-front/src/modules/app/components/AppRouter.tsx:6-18`
- `packages/twenty-front/src/modules/app/hooks/useCreateAppRouter.tsx:238-244`

**The Aha Moment:**
Feature flags become safe only when typed through route boundaries.

**Socratic Checkpoint:**
1. Which component derives the flags?
2. Which consumes them?
3. What if type changed to `string`?
4. Where is default behavior if flags undefined?
5. Why pass through route layer not global singleton?

**How to self-grade:**
Describe origin, boundary, and usage of both flags.

**Connects To:**
Mission 9.

### Mission 7: Following Data Into the App
**Tier:** Junior
**Time Estimate:** 40 minutes
**Goal:** Follow user data from backend response into state slices.

**The Concept:**
Hydration is controlled fan-out from one response to many consumers.

**Design Intent Before You Read the Code:**
Use current user loading hook.

**Find It In The Code:**
Open `packages/twenty-front/src/modules/users/hooks/useLoadCurrentUser.ts:47-135`.

**The Aha Moment:**
A single query can establish identity, permissions, locale, and workspace routing.

**Socratic Checkpoint:**
1. Why `fetchPolicy: 'network-only'`?
2. Which states are updated first and why?
3. Where does locale activation happen?
4. What if `currentWorkspace` is null?
5. Which update is critical for permission gating?

**How to self-grade:**
Show state fan-out order and one downstream consumer.

**Connects To:**
Mission 9, Mission 14.

### Mission 8: Navigation Is the App's Skeleton
**Tier:** Junior
**Time Estimate:** 35 minutes
**Goal:** Explain route map structure and URL entry behavior.

**The Concept:**
Routing is UI topology.

**Design Intent Before You Read the Code:**
Read central route factory.

**Find It In The Code:**
Open `packages/twenty-front/src/modules/app/hooks/useCreateAppRouter.tsx:109-267`.

**The Aha Moment:**
Route trees encode authorization, onboarding, and layout strategy in one graph.

**Socratic Checkpoint:**
1. What are default layout routes?
2. What are blank layout routes?
3. Where are settings routes delegated?
4. Which route catches unknown paths?
5. What happens on direct URL hit?

**How to self-grade:**
Redraw route tree and explain at least three branches.

**Connects To:**
Mission 12, Mission 14.
