# Fake-Code Contrasts

Eight bad-vs-better pairs. **All snippets below are illustrative fake code: not from this repo.** Each pair exposes a novice instinct, shows the better shape, and points to where Twenty does it right.

---

## Contrast 1 — Authorization in the resolver (and only there)

```ts
// Illustrative fake code: not from this repo — BAD
async companies(user: User) {
  const rows = await db.query('SELECT * FROM company');
  return rows.filter(r => user.canSee(r)); // post-hoc, breaks counts/pagination
}
```

```ts
// Illustrative fake code: not from this repo — BETTER
async companies(ctx: AuthContext) {
  return companyRepo.forContext(ctx).findMany(); // repo injects predicates into SQL
}
```

Real anchor: predicates applied inside the query builder so pagination/counts stay truthful — [workspace-select-query-builder.ts#L369-L390](../../../packages/twenty-server/src/engine/twenty-orm/repository/workspace-select-query-builder.ts#L369-L390). Novice instinct exposed: filtering after fetching.

## Contrast 2 — Coupling UI shape to DB shape

```ts
// Illustrative fake code: not from this repo — BAD
<td>{record.name_first + ' ' + record.name_last /* DB column names in JSX */}</td>
```

```ts
// Illustrative fake code: not from this repo — BETTER
const field = fieldDefinition(metadata, 'name'); // composite FULL_NAME type
<FieldDisplay definition={field} value={record.name} />
```

Real anchor: the front renders through field *definitions* with typed guards per field type ([usePersistField.ts#L97-L182](../../../packages/twenty-front/src/modules/object-record/record-field/ui/hooks/usePersistField.ts#L97-L182)) — the UI knows field *types*, not column layouts. Payoff: custom fields render with zero UI changes.

## Contrast 3 — Missing tenant scoping (IDOR)

```ts
// Illustrative fake code: not from this repo — BAD
app.get('/api/notes/:id', (req, res) => res.json(db.notes.findById(req.params.id)));
```

```ts
// Illustrative fake code: not from this repo — BETTER
app.get('/api/notes/:id', auth, (req, res) =>
  res.json(db.forWorkspace(req.workspace.id).notes.findById(req.params.id)));
```

Real anchor: the datasource itself is workspace-scoped before any query runs ([common-base-query-runner.service.ts#L144-L153](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-base-query-runner.service.ts#L144-L153)). Structural scoping beats per-endpoint discipline.

## Contrast 4 — Pessimistic save UX

```ts
// Illustrative fake code: not from this repo — BAD
const save = async () => { setSpinner(true); const r = await api.create(input); setSpinner(false); addToList(r); };
```

```ts
// Illustrative fake code: not from this repo — BETTER
const save = async () => {
  const id = uuid();
  cache.insert({ id, ...input });            // instant
  try { await api.create({ id, ...input }); } // same identity
  catch (e) { cache.remove(id); throw e; }    // compensate
};
```

Real anchor: [useCreateOneRecord.ts#L87-L188](../../../packages/twenty-front/src/modules/object-record/hooks/useCreateOneRecord.ts#L87-L188). The client-minted id is the trick juniors miss.

## Contrast 5 — N+1 relations

```ts
// Illustrative fake code: not from this repo — BAD
const companies = await repo.find();
for (const c of companies) c.people = await peopleRepo.find({ companyId: c.id }); // N queries
```

```ts
// Illustrative fake code: not from this repo — BETTER
const companies = await repo.find({ relations: { people: true } }); // join/batch once
```

Real anchor: relation loading is handled centrally with depth limits — nested 1:N-in-1:N is outright rejected ([common-base-query-runner.service.ts#L406-L414](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-base-query-runner.service.ts#L406-L414)); the API *shape* prevents the worst N+1s rather than trusting every caller.

## Contrast 6 — Side effects before the write commits

```ts
// Illustrative fake code: not from this repo — BAD
await sendWelcomeEmail(user.email);   // side effect first
await db.insert(user);                // insert fails → email already sent
```

```ts
// Illustrative fake code: not from this repo — BETTER
await db.transaction(async tx => {
  await tx.insert(user);
  await tx.insert(outbox, { type: 'user.created', payload: user }); // same tx
});
// relay delivers from outbox with retries
```

Real anchor (and honest caveat): Twenty emits *after* the SQL execute ([workspace-insert-query-builder.ts#L224-L275](../../../packages/twenty-server/src/engine/twenty-orm/repository/workspace-insert-query-builder.ts#L224-L275)) — better than the BAD shape, but without an outbox; see [03-architecture-and-patterns/04](../03-architecture-and-patterns/04-side-effects-async-and-reliability.md) for why BETTER is still stronger.

## Contrast 7 — Swallowed errors without a policy

```ts
// Illustrative fake code: not from this repo — BAD
try { await charge(customer); } catch (e) { console.log(e); } // payment maybe happened?
```

```ts
// Illustrative fake code: not from this repo — BETTER
try { await charge(customer); }
catch (e) {
  await audit.record('charge.failed', { customer, error: e }); // durable evidence
  throw e;                                                     // control flow decides retry
}
```

Real anchor: the two policies compared — webhook swallow-with-audit ([call-webhook.job.ts#L102-L116](../../../packages/twenty-server/src/engine/metadata-modules/webhook/jobs/call-webhook.job.ts#L102-L116), defensible for telemetry-grade effects) vs workflow record-and-rethrow ([run-workflow.job.ts#L63-L73](../../../packages/twenty-server/src/modules/workflow/workflow-runner/jobs/run-workflow.job.ts#L63-L73), required for business effects). The BAD shape is bad because it *chose neither*.

## Contrast 8 — `any` at the boundary

```ts
// Illustrative fake code: not from this repo — BAD
function persist(value: any) { db.save(value.field.trim()); } // compiles; explodes on numbers
```

```ts
// Illustrative fake code: not from this repo — BETTER
function persist(value: unknown) {
  if (!isTextValue(value)) throw new InvalidInput();
  db.save(value.field.trim()); // narrowed, proven
}
```

Real anchor: `valueToPersist: unknown` + guard pairs ([usePersistField.ts#L85-L182](../../../packages/twenty-front/src/modules/object-record/record-field/ui/hooks/usePersistField.ts#L85-L182)); repo bans `any` entirely ([CLAUDE.md](../../../CLAUDE.md)).

---

Drill: for two contrasts of your choice, find one *more* place in the repo that follows the BETTER shape, and one place in any past project of yours that followed the BAD shape — write the one-paragraph war story. That paragraph is a behavioral-interview asset ([08-interview-prep/06](../08-interview-prep/06-behavioral-star-stories.md)).
