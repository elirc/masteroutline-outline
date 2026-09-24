# Refactor and Design Katas

Eight judgment exercises. Output is words and diffs-on-a-fork, not upstream PRs. Each has self-grading criteria; most take 1–3 hours. These are the highest-density senior training in the curriculum.

## K-1: Boundary-leak repair (paper refactor)
Take the inline query assembly in [documents.list L119–171](../../server/routes/api/documents/documents.ts#L119-L171). Design (don't fully implement) its extraction: what layer does it belong to (command? model static? query-object helper?), what's its testable interface, what stays in the route.
Grade: Strong = your design keeps HTTP params out of the extracted unit's signature *and* you wrote one paragraph on why you might still choose to leave it inline (locality argument) — a kata isn't passed until you can argue the other side.

## K-2: Outbox RFC
Write the actual RFC for [SP-1](03-senior-build-projects.md) using the template in [07-career/02-writing-prs-and-rfcs.md](../07-career-and-collaboration/02-writing-prs-and-rfcs.md): context, problem evidence, proposal, alternatives (accept-loss, Kafka, listen/notify), migration, risks, open questions.
Grade: Strong = alternatives section is genuinely tempting (a reader could disagree), and the "do nothing" option is costed honestly.

## K-3: Split a module without breaking anyone
Plan the mechanical split of [documents.ts](../../server/routes/api/documents/documents.ts) (2,106 lines) into ~5 files with zero behavior change. Deliverable: the file manifest, what's shared (imports, helpers), PR sequencing to keep each reviewable, and the *proof strategy* that nothing changed (route-table snapshot? test inventory?).
Grade: Strong = your plan survives "two other PRs merged into documents.ts mid-split" — you sequenced to minimize conflict windows.

## K-4: Remove duplication — or defend it
Find two similar-looking recipient-computation passages among the notification tasks ([server/queues/tasks](../../server/queues/tasks) — e.g. the Document*Notifications family). Decide: extract or leave? Write the decision memo.
Grade: Strong = you applied "duplication is cheaper than the wrong abstraction" with *specific* divergence risks (each task's recipient rules evolve independently), not as a slogan.

## K-5: Type-safety upgrade
Design the typed presenter contract ([SP-2](03-senior-build-projects.md)) for exactly one presenter ([share.ts](../../server/presenters/share.ts) is small). Implement it on a fork. Measure: how many call sites changed, what did the compiler catch?
Grade: Strong = you hit a versioned-shape problem (`x-api-version`) and made a reasoned call (union vs separate presenters), documented in a comment.

## K-6: Design a migration under fire
Scenario: the `events.data` → `changes` column migration ([Event.ts L59–73](../../server/models/Event.ts#L59-L73)) must complete: `data` becomes *only* arbitrary event payload, `changes` becomes *only* model diffs, and clients depend on `data` today. Plan the full expand/migrate/contract sequence including the client releases.
Grade: Strong = your plan has a step where *both* columns are written, a verification query gating the contract step, and an explicit "how long do we wait for old clients" answer with a mechanism (version headers exist — use them).

## K-7: Reduce an N+1 you first have to find
Instrument any list endpoint (Sequelize logging) until you find a per-row query — presenter internals are the likely spot (e.g., per-document `isCollectionDeleted` at [presenter L107](../../server/presenters/document.ts#L105-L112) — does it query per doc in list contexts? *investigate*). Fix on a fork with a before/after query count.
Grade: Strong = the evidence sentence ("N queries → 2 for a 50-item list, verified by log capture") plus honesty if your suspect turned out innocent.

## K-8: Review a flawed design (not code)
A teammate proposes: "cache `presentDocument` output in Redis keyed by document id, invalidated on `documents.update` events, to speed up lists." Write the review.
Expected findings: the payload is *viewer-dependent* (isPublic, memberships, `lastViewedAt` — [presenter L88–129](../../server/presenters/document.ts#L88-L129)) so a per-document key serves wrong data across users (a **correctness** bug wearing a perf costume); invalidation via events inherits the event-loss gap; and the real perf problem (if any) is unmeasured.
Grade: Strong = you led with the correctness kill-shot, proposed the narrower valid version (cache only viewer-independent expensive sub-computations), and asked for the measurement first.

---

Meta-rubric across katas: each verdict cites file evidence; each "no" comes with the narrower "yes"; each abstraction argument names concrete divergence/convergence forces. Collect your eight memos — they are direct feedstock for [interview stories](../08-interview-prep/06-behavioral-star-stories.md) and the strongest portfolio artifact this curriculum produces.
