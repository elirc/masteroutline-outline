# Maintainer Communication

Open source maintainers (and senior teammates — same skills) triage dozens of low-effort messages daily. Every template below follows one principle: **do the work first, then ask the smallest possible question.** Asking for help without outsourcing your thinking is the difference between "promising contributor" and "support burden."

## Asking a question

```
I'm trying to [goal]. I've read [files, with paths] and understand [correct summary].
What I can't reconcile: [specific confusion, ≤2 sentences].
My current best guess: [guess]. Is that right, or am I missing something?
```

The "best guess" line is load-bearing: it converts an open question into a yes/no, shows your reasoning for correction, and often answers itself as you write it (the rubber-duck effect is not optional — it's the point).

## Reporting a bug

```
**What happens:** [observed, exact]
**What I expected:** [expected, and why — cite docs/code if the expectation comes from there]
**Repro:** [numbered steps from a clean state; smallest version that still fails]
**Environment:** [version/commit, self-hosted vs cloud, browser/OS if relevant]
**Evidence:** [log lines, screenshot, failing test if you wrote one]
**Where I think it lives (optional):** [file:line hypothesis, labeled as hypothesis]
```

A failing test as the repro is the gold standard — this repo's factory system ([server/test/factories.ts](../../server/test/factories.ts)) makes most backend repros ~15 lines. A hypothesis labeled as one is welcome; a hypothesis asserted as fact ("your auth is broken") is how reports get closed.

## Proposing a feature

```
**Problem** (not solution): who hits it, how often, current workaround.
**Proposal sketch:** smallest version; how it follows existing patterns [anchors].
**Willing to build it?** yes/no, and your rough plan if yes.
**Prior art:** existing issues/PRs you searched (link them — proves you looked).
```

For this specific repo: it's a commercial open-source product (BSL license) with its own roadmap — expect and respect "not planned" outcomes; the search-first habit prevents most rejections.

## Responding to review feedback

Covered in [01-code-review-mindset.md](01-code-review-mindset.md); the maintainer-flavored additions: batch your responses (one pass, all comments), push commits *after* responding so the reviewer reads intent before diff, and thank specifically, not performatively ("the transaction-threading catch would have been a prod bug" — specific gratitude teaches you and rewards them).

## Disagreeing respectfully

Sequence: (1) restate their position until they'd endorse the restatement; (2) your evidence, anchored; (3) the decision rule you propose ("if we expect >X, yours wins; below it, mine"); (4) accept the call and move. Escalate *decisions*, never *people*: "can we get a second opinion on the tradeoff" not "they're wrong."

Timing note: maintainers respond in days, not minutes. Batch questions, keep working on parallel threads while blocked, and never ping-bump inside a week.

## Interview angle

- *"Tell me about working with people you've never met"* / *"how do you unblock yourself?"* — the ask-template as your method: read → summarize → guess → smallest question. Interviewers hear "self-directed but knows when to ask" — precisely the mid-level bar.
- *"Tell me about receiving hard feedback"* — the respond-to-review protocol as your story structure.

## Drill

Write (don't send) a real bug report for anything you found during the [debugging scenarios](../05-quality-engineering/03-systematic-debugging.md) or an *investigate* item from the [risk register](../09-reference/risk-register.md) — e.g., the event-loss window — using the template. Grade: could a maintainer act on it without asking you anything? That's the bar.
