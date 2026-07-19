# Learning Rubrics

Observable behaviors, not vibes. Self-assess honestly; "interview-ready" is the bar for saying it confidently in a loop.

## Codebase navigation

| Level | Observable behavior | Interview-ready when… |
| --- | --- | --- |
| Junior | Finds a named file/symbol with grep in minutes; explains one flow with the docs open | You can name the six flows and pick the right one for a question |
| Mid | Locates the *enforcement point* for a behavior (where permissions/validation actually happen) without docs | You cite file/behavior pairs from memory ("the ORM builders validate on every execute path") |
| Senior | Predicts where a feature's code *must* live from the architecture, then confirms | You can say where a new feature goes and what pattern it follows before opening the editor |

## JS/TS/Node

| Level | Observable | Interview-ready when… |
| --- | --- | --- |
| Junior | Correct async code; basic types | Definitions correct in one sentence |
| Mid | Chooses serial vs parallel with reasons; `unknown`+guards at boundaries; explains a stale closure they fixed | Every answer includes a real example unprompted (the four-beat structure) |
| Senior | Designs error-handling *policies* per boundary; knows where static typing ends and runtime contracts begin | You can argue both sides of swallow-vs-rethrow with cases |

## Architecture & data

| Level | Observable | Interview-ready when… |
| --- | --- | --- |
| Junior | Draws the layer stack; knows tenant isolation exists | Can define invariant/boundary/idempotency in context |
| Mid | Explains the custom-fields storage tradeoff table; picks correctly for a given constraint set | 45-min design covers API, data model, permissions, async with one tradeoff each |
| Senior | Surfaces delivery-guarantee and migration-rollback decisions unprompted; proposes phased rollouts with abort criteria | Your design review of your own proposal finds its weakest point first |

## Quality engineering

| Level | Observable | Interview-ready when… |
| --- | --- | --- |
| Junior | Writes passing tests for the happy path; runs the right commands | Can say what layer a given test belongs to and why |
| Mid | Tests the denied/failure case by default; debugs by layer-elimination narrated aloud | Timed rounds D1–D3 at "Solid" rubric |
| Senior | Adds the regression test that forces a *decision* (Ticket 1); writes "we'll know it broke because ___" for every change | You instrument before you optimize and can quote a number |

## Collaboration

| Level | Observable | Interview-ready when… |
| --- | --- | --- |
| Junior | PRs with what/why/how-tested; responds to every review comment | Two STAR stories under 2 minutes |
| Mid | Tiered review findings; small-slice proposals; evidence-first questions | Stories 1/5/7 rehearsed with real outcomes |
| Senior | RFCs with alternatives tables and abort stories; concedes fast when wrong | You have one "I recommended against my own idea" story |

## Self-assessment protocol

Score each table Junior/Mid/Senior monthly. Evidence required: a file you can open, a test you wrote, a recording of a timed round — no evidence, no credit. The Day-7/Day-14 checkpoints in the [cram plan](../08-interview-prep/07-two-week-cram-plan.md) map onto the "interview-ready" column.
