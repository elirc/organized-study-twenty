# Code Review Mindset

## The five layers (review in this order)

1. **Does it work?** — the stated behavior, exercised how? (Ask for evidence, don't re-derive it.)
2. **Is it correct?** — edge cases, failure paths, concurrency. In this repo: the denied case, the API-key case, the worker-process case.
3. **Will it stay correct?** — tests pinning the behavior; invariants stated where enforced.
4. **Does it fit?** — follows an existing pattern or justifies a new one. Twenty has strong conventions ([CLAUDE.md](../../../CLAUDE.md)): named exports, types over interfaces, no `any`, functional components, kebab-case files.
5. **Is it kind to future maintainers?** — names, comment quality (WHY not WHAT), diff size.

Block only on layers 1–3 (and security anywhere). Layers 4–5 are "Important/Optional" unless the pattern violation creates a trap.

## Twenty-specific review checklist

- [ ] New/changed query path: does it go through the runner pipeline / permission-enforcing builders, or around them? Anything around them is a security conversation ([03-architecture-and-patterns/03](../03-architecture-and-patterns/03-validation-auth-and-permissions.md)).
- [ ] `shouldBypassPermissionChecks: true` anywhere → justification required.
- [ ] Public contract touched (GraphQL SDL, REST shape, webhook payload)? → downstream dependents updated ([call-webhook-jobs.job.ts#L34-L37](../../../packages/twenty-server/src/engine/metadata-modules/webhook/jobs/call-webhook-jobs.job.ts#L34-L37)), breaking-change CI consulted.
- [ ] Entity change → instance command with `up` **and** `down`; committed command logic never rewritten ([CLAUDE.md](../../../CLAUDE.md)).
- [ ] New async job → explicit swallow-vs-rethrow decision, timeout on any network call, semantics documented (the webhook/workflow contrast is the house reference: [03-architecture-and-patterns/04](../03-architecture-and-patterns/04-side-effects-async-and-reliability.md)).
- [ ] Front state → correct tier (Apollo cache / Jotai atom-family / local); no broad atoms feeding table cells.
- [ ] GraphQL change → `graphql:generate` ran; schema backward compatible.
- [ ] Tests at the right layer (would this test pass with the DB deleted? then it can't verify SQL/permissions).

## Comment style — specific, kind, actionable

Weak: "This is inefficient."
Strong: "This queries per row (N+1) — 100 rows means 100 requests. The row query already supports depth-1 relations via `useGenerateDepthRecordGqlFieldsFromObject`; could we extend that instead? Happy to pair if useful."

Weak: "Why no tests?"
Strong: "The denied-role case is the risky one here — a one-case integration test next to `auth.integration-spec.ts` would pin it. See Recipe 3 pattern."

Tone rules: review the code, not the author; ask questions where you're <90% sure; state severity explicitly (blocking / important / optional / nit) so the author can prioritize; approve with nits instead of holding approval hostage.

Practice: the eight katas in [04-code-reading-gym/04-review-katas.md](../04-code-reading-gym/04-review-katas.md) with the severity discipline above. Interview cross-link: review rounds in [08-interview-prep/05](../08-interview-prep/05-debugging-and-code-review-rounds.md).
