# Writing PRs & RFCs

## PR anatomy (make "yes" easy)

```markdown
## What
One paragraph: the behavior change, user-visible first.

## Why
The problem/ticket; why now; why this approach over the obvious alternative (one sentence each).

## How tested
- [ ] npx nx typecheck twenty-server / twenty-front
- [ ] npx nx lint:diff-with-main <pkg>
- [ ] targeted jest: <command + result>
- [ ] integration/e2e where SQL/permissions/UI in blast radius
Evidence beats assertion: paste the failing-then-passing test name, or a screenshot for UI.

## Risks
Blast radius (which module/contract), failure mode if wrong, and the rollback story
(flag? revert? instance-command down?).

## Follow-ups
What you deliberately did NOT do, so reviewers don't relitigate scope.
```

Sizing: one logical change per PR. If your diff touches a public contract *and* refactors internals, split it — contract diffs need contract-level review attention ([Kata 8's lesson](../04-code-reading-gym/04-review-katas.md)).

Commit messages: imperative subject ≤72 chars ("Add failure metric to webhook job"), body = why + notable tradeoffs. The subject should make sense in `git log --oneline` two years later.

## When to RFC

Write an RFC (before code) when any of: a public contract changes shape; a new infrastructure dependency; delivery semantics change (M1!); a pattern is introduced that others will copy; rollback is nontrivial. Skip the RFC for anything a feature flag + small diff can safely prove.

## RFC template (tailored to this repo)

```markdown
# RFC: <title>

## Problem
Current behavior with file anchors (e.g. call-webhook.job.ts#L102-L116 swallows failures).
Who is affected and how badly. Confirmed facts vs hypotheses, labeled.

## Goals / Non-goals
Non-goals prevent scope creep and reviewer tangents.

## Proposed design
Diagram + the 2–3 real decisions with alternatives table:
| Decision | Chosen | Alternative | Why |

## Contract & compatibility
GraphQL/REST/webhook payload impact; Zapier/SDK/OpenAPI updates; breaking-changes CI.

## Migration & rollout
Phases, flags per phase, abort story per phase, mixed-version window behavior.

## Testing & observability
The test that proves the property; the metric/audit that detects regression in prod
("we'll know it broke because ___").

## Open questions
Genuinely open — not rhetorical.
```

Model content: [06-contribution-practice/03-senior-build-projects.md](../06-contribution-practice/03-senior-build-projects.md) Project 1 is an RFC skeleton for the outbox; Kata 2 asks you to complete it.

Interview angle: "How do you drive a technical decision across a team?" — describe the RFC flow with the M1 example: evidence (anchored behavior) → options table → phased rollout with abort stories. That answer structure works even at companies with no RFC culture.
