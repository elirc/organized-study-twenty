# Maintainer Communication

Principle: **bring evidence, not homework.** Every message should show what you already did, so the reader adds only what they uniquely know.

## Asking a question

```markdown
**Context:** Implementing retry for webhook delivery (issue #NNN).
**What I found:** call-webhook.job.ts catches all errors without rethrow (L102-L116),
so the retryLimit at enqueue (call-webhook-jobs.job.ts#L72) never engages for HTTP failures.
**My read:** this makes delivery at-most-once, single attempt. Audit records the failure.
**Question:** is that intentional (avoid hammering broken receivers) or an oversight?
If intentional I'll document it; if not, I have a per-delivery retry design to propose.
```

Why it works: anchored facts, your interpretation labeled as interpretation, and both branches pre-planned — the maintainer answers in one line either way.

## Reporting a bug

Minimal repro or it didn't happen. Template: **environment** (commit, setup path — e.g. `setup-dev-env.sh`, Node version), **steps** (numbered, from clean state), **expected vs actual** (with exact error text), **narrowing already done** (layers eliminated, per [05-quality-engineering/03](../05-quality-engineering/03-systematic-debugging.md)), **suspected area** (anchors, labeled as suspicion). If you can't reproduce it twice, say so and file it as intermittent with the evidence you have — honesty about uncertainty builds more trust than confident noise.

## Proposing a feature

Lead with the user problem and the smallest slice: "Webhook consumers can't distinguish receiver outages from unsubscribed events. Smallest useful slice: failure metrics + settings visibility (2–3 days, additive). Larger follow-up: retries (RFC attached). Would you accept the small slice while the RFC is discussed?" Small-slice-first proposals get merged; big-bang proposals get parked.

## Responding to review

- Every comment gets a response: a diff, a question, or a reasoned disagreement — never silence.
- Disagreeing: restate their concern first ("You're right that per-delivery jobs 20x queue volume"), then evidence ("at current event rates that's ~N jobs/min vs Redis capacity M"), then offer the decision back ("happy to gate on a load test if that helps").
- When you're wrong, say "good catch, fixed in <commit>" and move on. Speed of concession is a seniority signal, not a weakness.
- Batch pushes; don't force reviewers to re-review per nit fix.

## Open-source etiquette specific to a repo like Twenty

Check for an existing issue/PR before opening either; conventions live in [CLAUDE.md](../../../CLAUDE.md) and `.cursor/rules/` — read them before your first PR, cite them in reviews instead of arguing taste; keep external-contributor PRs small (there's an [auto-draft workflow for external PRs](../../../.github/workflows/external-contributor-pr-auto-draft.yaml) — expect process); never rewrite committed instance-command logic (house rule with data-loss reasons behind it).

Interview angle: "Tell me about working with a team you didn't control" — open-source contribution stories are the strongest version of this answer, and every template above becomes a beat in that story. ([08-interview-prep/06](../08-interview-prep/06-behavioral-star-stories.md), Story 7.)
