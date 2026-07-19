# Type System & Contracts

TypeScript's job is to make illegal states unrepresentable *at compile time*. But this repo's core feature — user-defined schemas — means the most important types exist only *at runtime*. Watching how Twenty manages that tension is the best TS education in the repo.

## Compile-time contracts

- **Generics with intent.** The base runner is `CommonBaseQueryRunnerService<Args extends CommonQueryArgs, Output extends CommonQueryResult>` ([common-base-query-runner.service.ts#L60-L63](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-base-query-runner.service.ts#L60-L63)); each subclass pins its arg and result types ([common-create-one-query-runner.service.ts#L24-L27](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-create-one-query-runner.service.ts#L24-L27)). Transferable: generics on a base class let the *pipeline* be written once while every operation keeps precise types. Note the repo convention: descriptive parameter names (`TData`), never bare `T` ([CLAUDE.md](../../../CLAUDE.md)).
- **Union + type-guard narrowing as the workhorse.** `usePersistField` dispatches on ~20 field types purely through paired guards: `isFieldText(fieldDefinition) && isFieldTextValue(valueToPersist)` ([usePersistField.ts#L97-L182](../../../packages/twenty-front/src/modules/object-record/record-field/ui/hooks/usePersistField.ts#L97-L182)). Each guard narrows both the definition and the value. Failure mode is visible in the same file: a type nobody added to the chain throws at runtime (L282-L290) — the compiler cannot force exhaustiveness over this shape. A `switch` on a discriminant with a `never` check would; that's a legitimate critique to raise in an interview.
- **Auth context unions.** One request context may be a user, API key, or application; guard functions like `isUserAuthContext` / `isApiKeyAuthContext` narrow before use ([common-base-query-runner.service.ts#L292-L302](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-base-query-runner.service.ts#L292-L302)). This is the "discriminated union + guards" pattern applied to security — narrowing *is* the authorization branch.
- **`unknown` at the boundary.** `valueToPersist: unknown` ([usePersistField.ts#L85](../../../packages/twenty-front/src/modules/object-record/record-field/ui/hooks/usePersistField.ts#L85)) forces every consumer through a guard. `any` would compile identically and verify nothing — the repo bans `any` outright ([CLAUDE.md](../../../CLAUDE.md)). Interview one-liner: "`unknown` is `any` that makes you prove it."
- **Shared helpers over ad-hoc guards.** `isDefined()` from `twenty-shared/utils` everywhere (e.g. [workspace-schema.factory.ts#L33](../../../packages/twenty-server/src/engine/api/graphql/workspace-schema.factory.ts#L33)) — one narrowing helper, consistent semantics for `null | undefined`.

## Runtime contracts (where TS can't reach)

- **The GraphQL schema is data.** SDL is computed from metadata and compiled with `makeExecutableSchema` ([workspace-schema.factory.ts#L48-L67](../../../packages/twenty-server/src/engine/api/graphql/workspace-schema.factory.ts#L48-L67)). No TS type knows what fields `company` has — that's why records flow as `ObjectRecord` (a loose map type) and correctness is enforced by *metadata validation*, arg processors, and the parser ([graphql-query.parser.ts](../../../packages/twenty-server/src/engine/api/graphql/graphql-query-runner/graphql-query-parsers/graphql-query.parser.ts)).
- **Codegen bridges the gap for known objects.** The front generates typed operations (`npx nx run twenty-front:graphql:generate`, __inferred__) into `src/generated*`; custom objects still go through generic record types. Transferable: codegen gives you compile-time safety *for the part of the contract that's static*; design explicitly for the dynamic remainder.
- **Metadata version as a contract version.** Every request binds a metadata version ([jwt-auth.guard.ts#L29-L33](../../../packages/twenty-server/src/engine/guards/jwt-auth.guard.ts#L29-L33)); schema caches key off it. That's semantic versioning applied per-tenant, per-request.

## Pitfall checklist

- [ ] `as` casts to silence the compiler where a guard belongs (the repo does carry some — e.g. `as CommonExtendedInput<Args>` at [common-base-query-runner.service.ts#L133-L136](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-base-query-runner.service.ts#L133-L136); each one is a small hole in the proof).
- [ ] Non-exhaustive guard chains for growing unions — prefer discriminated unions + `never` checks.
- [ ] Trusting client-provided shapes: everything crossing the wire is `unknown` until validated.
- [ ] Types-only changes to a *runtime* contract (GraphQL/REST/webhook payloads) — those need migration thinking, not just compiler happiness.

## Drills

1. Sketch (on paper) how you'd refactor `usePersistField`'s guard chain into a discriminated-union `switch` with compile-time exhaustiveness. What breaks? Strong: you notice the *pairing* of definition-guard and value-guard is the hard part — the discriminant lives on the definition, but the value needs independent validation.
2. Find three `as` casts in [common-base-query-runner.service.ts](../../../packages/twenty-server/src/engine/api/common/common-query-runners/common-base-query-runner.service.ts) and classify each: unavoidable (generic erasure), convenience, or risk.

## Interview angle

- "unknown vs any" → [08-interview-prep/01](../08-interview-prep/01-js-ts-node-deep-dive.md) Q8
- "Type narrowing / discriminated unions" → Q9
- "Generics: when and how" → Q10
- "How do you type a system whose schema is user-defined?" → [08-interview-prep/04](../08-interview-prep/04-system-design-from-this-repo.md), step 3
