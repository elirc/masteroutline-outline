# Code Review Mindset

## The five layers (review in this order)

1. **Does it work?** — the happy path does what the PR claims. (Cheapest to verify: run it, or trust the test.)
2. **Is it correct?** — edge cases, concurrency, permissions, failure paths. This is where your [key-flows](../01-codebase-cartography/05-key-flows.md) knowledge pays: you know where the bodies are buried (transactions unthreaded, events double-fired, policies unchecked).
3. **Will it stay correct?** — tests at the right layer, invariants made loud (invariant/type/constraint), no time bombs (unbounded growth, missing retention).
4. **Does it fit?** — house patterns: would a maintainer recognize this as Outline code? (Middleware chain shape, command placement, presenter discipline, factory-based tests.)
5. **Is it kind to future maintainers?** — naming, JSDoc per [AGENTS.md](../../AGENTS.md#L143-L152), no cleverness without a comment paying its rent.

Severity discipline: **Blocking** = correctness/security/contract breakage. **Important** = will cause pain, fixable now cheaply. **Optional/nit** = preference; say so explicitly. Inflating nits to blockers costs you credibility exactly like deflating security issues does.

## Repo-specific review checklist

- [ ] Every new id in a request body authorized (`authorize` per object, scoped loads with `userId`) — see the [triple-check pattern](../../server/routes/api/documents/documents.ts#L1296-L1352)
- [ ] Mutating route has `transaction()`, and **every** DB call inside threads `ctx.state.transaction`
- [ ] New endpoint: Zod schema, rate-limit decision recorded, pagination on lists
- [ ] Events: created via ctx-aware helpers (audit fields), no PII in `data` (client-visible — [Event.ts L59–65](../../server/models/Event.ts#L59-L65))
- [ ] Async consumers idempotent (run-twice test present?)
- [ ] Presenter change = contract change: additive? public-share safe? client model updated?
- [ ] Client: business logic in stores not components; observer wrapping where observables are dereferenced
- [ ] Tests at the layer of the risk (authz → route test; logic → command test)
- [ ] Migration: `down` works, big-table operations `CONCURRENTLY`, boot-gate implications considered
- [ ] i18n: no hand-added translation strings ([AGENTS.md](../../AGENTS.md#L36-L38)); user-facing strings wrapped for extraction

## Comment phrasing that works

The formula: **observation (with anchor) → consequence → suggestion → severity.**

> **[blocking]** `documents.wordcount` loads the doc without `{ userId }` and never calls `authorize` — any team member can read any document by id (IDOR). Suggest the `documents.info` pattern (documents.ts L574): scoped `findByPk` + `authorize(user, "read", document)`.

> **[important]** This `findAll` inside the `transaction()`-wrapped handler doesn't pass `transaction`, so it reads outside the txn and can see pre-transaction state. Thread `ctx.state.transaction` like the sibling call above.

> **[nit, optional]** House style puts multi-model orchestration in `server/commands/` — fine here given the size, but if this grows a third model write, let's extract.

And the two non-code moves that senior reviewers make: (1) *praise specifically* ("the run-twice idempotency test is exactly right") — it teaches as much as criticism; (2) *review the tests first* — they tell you what the author thinks the change does, which frames everything.

## When you're the author

Respond to every comment (fix, push back with evidence, or explicitly defer with a tracked follow-up). Push-back template: "I considered that — the reason I chose X is [evidence/anchor]; happy to switch if you weigh [tradeoff] differently." Never silently ignore; never silently comply against your judgment either — both erode the review's value.

## Interview angle

- *"Walk me through how you review a PR"* — the five layers, in order, with the tests-first move.
- *"A senior disagrees with your review comment"* — severity honesty + evidence-based pushback + "disagree and commit" for non-blockers.
- Live review rounds: [08-interview-prep/05](../08-interview-prep/05-debugging-and-code-review-rounds.md) has timed kata versions.

## Drill

Review a real merged PR: pick a recent commit from `git log --oneline -20` (e.g. the fast-click fix `3524427c4`), view it with `git show`, and write the review you'd have given *before* reading any linked discussion. Then check: did the actual diff include tests/handling you'd have asked for? Two rounds of this calibrates you faster than any reading.
