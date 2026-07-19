# 03 — Frontend apprenticeship: twenty-front without tears

The server packs (fabledocs2/3) barely touch `packages/twenty-front`, and a
junior on this team will ship frontend work in week one. This is the minimum
mental model plus a guided slice.

## The three-sentence mental model

The frontend is a **metadata-driven rendering engine**, mirroring the server:
it doesn't know what a "Company" is at build time either. It fetches object
metadata, generates queries against the runtime GraphQL schema (ADR-006), and
renders generic record components (tables, boards, show pages) configured by
that metadata. Your job is almost never "build the Listing page" — it's
"extend the generic machinery so *all* objects get the capability."

## Layout that matters

- `src/modules/object-metadata/` — client-side metadata: hooks, `states/`
  (Jotai), types. The frontend twin of the server's metadata modules.
- `src/modules/object-record/` — the generic record machinery. Learn these
  subtrees in order: `record-store/` (canonical client-side record state),
  `record-field/` (field type rendering/editing), `record-table/`,
  `record-index/`, `record-filter/` + `advanced-filter/`, `cache/` (Apollo
  cache interplay).
- `src/modules/ui/` — the layout/input/display primitives; check here before
  writing any new visual component (also check `packages/twenty-ui`).
- `src/generated/` and `src/generated-metadata/` — codegen output. Never
  hand-edit; regenerate with `npx nx run twenty-front:graphql:generate`
  (add `--configuration=metadata` for the metadata schema).
- Conventions are enforced socially and by lint (CLAUDE.md): functional
  components only, named exports only, types over interfaces, no `any`,
  Linaria styled-components, state via **Jotai**.

## Jotai in this codebase, in five minutes

Global state is atoms; derived state is selectors; **dynamic collections are
atom families keyed by instance id** — that last one is the load-bearing
pattern, because there are N record tables and N fields on screen, each
needing isolated state. Look at any `states/` directory under
`object-record/record-store/` or `object-metadata/states/` and you'll find
the house style: small atom definitions, one per file, consumed via hooks
that wrap `useRecoilComponentValue`-style helpers. Rules that get PRs
rejected: no business logic inside components that belongs in hooks; state
updates via event handlers, not `useEffect` chains; functional updates
(`set(prev => ...)`).

**Exercise (30 min):** pick one atom family under `record-store/`, and trace:
who sets it (which hook, triggered by which user event), who reads it, and
what happens on unmount. Write the trace as five bullet points.

## Guided slice: add a read-only "field freshness" indicator

A contained end-to-end feature touching every layer you'll work in. Goal:
when hovering a field on the record show page, show how long ago that field's
record was updated (from the record's `updatedAt`).

1. **Find the render path.** Start at `record-show/`, descend to
   `record-inline-cell/` and `record-field/`. Identify the component that
   owns the hover state (there will already be one — extend, don't duplicate).
2. **Get the data.** `updatedAt` is already on every record in the store
   (`record-store/`) — no query change needed. Retrieve it via the existing
   record hook in scope; do **not** add a new fetch.
3. **Render.** A small styled (Linaria) tooltip element using an existing
   `ui/display` primitive. Match theme tokens; no hardcoded colors.
4. **Test.** A component test querying by visible text (Testing Library,
   user-event — house rules in CLAUDE.md), plus a Storybook story.
5. **Quality gates.** `npx nx lint:diff-with-main twenty-front`,
   `npx nx typecheck twenty-front`, run your test file directly.

**Exit questions:** Why did step 2 forbid a new fetch (what does duplicate
fetching do to the Apollo cache and the record store's consistency)? Where
would this feature's state live if it needed to persist across hovers — atom,
atom family, or component state, and why?

## The traps register (frontend edition)

- **Stale codegen**: type errors on fields you know exist → regenerate before
  debugging. (Same trap as the server test clients — ADR-008.)
- **Editing `generated*/`**: your change works until the next codegen run.
- **New component when a primitive exists**: check `ui/` and `twenty-ui`
  first; reviewers will.
- **`useEffect` for derived state**: derive with a selector or compute in
  render; effects are for external-world synchronization.
- **Metadata assumptions**: never hardcode object/field names outside the
  standard-object constants — the object you're rendering might be renamed,
  deleted, or custom. If you type the string `"company"` in a generic
  component, stop and reconsider.
