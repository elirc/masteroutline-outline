# Writing PRs and RFCs

## PR descriptions

The reviewer's questions, pre-answered — that's the whole art. Structure (matches how this repo's own commit subjects read: `fix:`/`feat:` conventional prefixes, per `git log`):

```
## What
One paragraph: the behavior change, user-visible first.

## Why
The problem/issue link. If subtle, the evidence (repro, log, measurement).

## How
Only the non-obvious: the approach chosen and the one you rejected, in two sentences.

## Tested
Exact commands + what they prove: "yarn test server/commands/documentUpdater.test.ts —
new cases: publish path, unchanged-content path."

## Risks / follow-ups
What could break, what you deferred (with a tracking note).
```

Rules that separate good from adequate: the *What* is readable by a PM; the *How* admits the rejected alternative (this pre-empts half of review debate); *Tested* names commands, not vibes ("tested locally" is a null statement); risks are yours to volunteer — a reviewer discovering an unmentioned risk costs more trust than the risk itself.

Commit messages: imperative subject ≤72 chars with the conventional prefix (`fix: Comments menu item does nothing when triggered from a document list` — real example from this repo's log); body = *why*, never a diff narration.

Keep PRs one-concern: the mechanical test is "can I write the What in one sentence without 'and'?" The [K-3 kata](../06-contribution-practice/04-refactor-and-design-katas.md)'s PR sequencing is this rule under pressure.

## When to RFC (vs just PR)

RFC when any of: contract changes (API/webhook/websocket payloads), schema changes beyond additive, new infrastructure (queue, cache, index engine), security-model changes, anything whose *rollback is hard*. The test: if being wrong costs more than a revert, write first, code second.

## RFC template (tailored to this repo)

```
# RFC: [title]

## Context
What exists today, with file anchors. (e.g. "Events enqueue post-commit — Event.ts L82–97;
no relay re-reads missed events.")

## Problem
Evidence it's real: incident, measurement, or a concrete failure scenario with likelihood.

## Proposal
The design. Include: schema deltas, new/changed contracts, which services touch it
(web/worker/websockets/collaboration — be explicit, they deploy as one but run separately),
and the migration sequence (expand → migrate → contract steps as numbered deploys).

## Alternatives
2–3, each with the reason it loses. Include "do nothing" with its cost.

## Risks and rollback
Blast radius per component; the rollback procedure as commands/steps, and what state
rollback does NOT restore.

## Test & observability plan
How we prove it works and how we'd know it broke in prod (metric/alert).

## Open questions
Numbered, each with an owner-shaped question ("needs a product decision on X").
```

Worked example to write as practice: the outbox RFC ([K-2](../06-contribution-practice/04-refactor-and-design-katas.md)). A strong RFC's tell: a reader can disagree *specifically* — vague proposals get vague feedback.

## Interview angle

- *"How do you communicate a technical proposal?"* — the RFC skeleton, plus "alternatives with honest costs" as the credibility mechanism.
- *"Tell me about a time you disagreed on direction"* — RFC-shaped answers (evidence → alternatives → decision → commit) score highest.
- Portfolio note: 2–3 well-written RFCs/design memos from module 06 are showable artifacts in interviews — often more impressive than the code.

## Drill

Take your MT-2 or MT-5 design note ([mid-level tickets](../06-contribution-practice/02-mid-level-feature-tickets.md)) and upgrade it to a full RFC with this template. Grade: every section non-empty, at least one alternative you'd genuinely half-prefer, rollback stated as steps. Time-box: 45 minutes — RFC writing must be cheap or you'll skip it when it matters.
