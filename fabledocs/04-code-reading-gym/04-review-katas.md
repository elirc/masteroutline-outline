# Review Katas

Nine fake PRs. For each: read the intent and diff summary, write your review (findings labeled **Blocking / Important / Optional**) *before* reading "Expected findings." Diffs are described, not shown in full — reviewing from a description forces you to ask the questions a real diff would answer. All scenarios are invented; the "files this resembles" anchors are real.

Review like this repo's maintainers would: kind, specific, with file references and a suggested path — see [07-career-and-collaboration/01-code-review-mindset.md](../07-career-and-collaboration/01-code-review-mindset.md).

---

## Kata 1: "Add documents.wordcount endpoint"
Author intent: new RPC returning word counts for a document.
Fake diff summary: adds `router.post("documents.wordcount", auth(), async (ctx) => { const doc = await Document.findByPk(ctx.request.body.id); ctx.body = { data: countWords(doc.text) }; })`.
Files this resembles: [documents.ts](../../server/routes/api/documents/documents.ts#L574-L652).
Expected findings — **Blocking:** no `validate()` schema (raw body access); `findByPk` without `{ userId }` + no `authorize(user, "read", doc)` → IDOR. **Important:** no null handling (`doc` may be null → crash); heavy sync `countWords` on a request path for big docs. **Optional:** naming (`documents.word_count` per house convention? check sibling names); should it live on the presenter/insights instead?
Good review comment example:
> This endpoint can read any document by id — we need the scoped load + policy check like documents.info does: `Document.findByPk(id, { userId: user.id })` then `authorize(user, "read", document)`. See documents.ts L574 for the pattern.

## Kata 2: "Fix flaky test by increasing timeout"
Author intent: a revision-processor test intermittently fails; PR bumps `testTimeout` to 30s.
Files this resembles: [RevisionsProcessor.test.ts](../../server/queues/processors/RevisionsProcessor.test.ts).
Expected findings — **Blocking:** timeout masks the race, doesn't fix it — demand the root cause (unawaited promise? Redis state shared between tests? ordering assumption?). **Important:** if the test polls, replace with awaiting the actual promise/job completion. **Optional:** flake tracking note.
The transferable rule: *a timeout bump is a confession, not a fix.*

## Kata 3: "Send Slack message when document published"
Author intent: adds a Slack API call inside `document.publish()` model method.
Files this resembles: [Document.ts](../../server/models/Document.ts), [plugins/slack](../../plugins/slack).
Expected findings — **Blocking:** external HTTP inside a model method = inside the request transaction (holds txn open; Slack outage breaks publishing). Must move to the event pipeline — a processor/task on `documents.publish`, in the Slack plugin, like other integrations. **Important:** retries/idempotency (duplicate Slack posts on retry?); team-level opt-in check. **Optional:** payload minimization.
This kata is the side-effect-placement lesson in [03-architecture/04](../03-architecture-and-patterns/04-side-effects-async-and-reliability.md) as a review scenario.

## Kata 4: "Show archived documents in search results"
Author intent: adds `statusFilter` default including archived.
Fake diff summary: changes the schema default in `documents/schema.ts` from `undefined` to `[StatusFilter.Published, StatusFilter.Archived]`.
Files this resembles: [documents/schema.ts](../../server/routes/api/documents/schema.ts), [documents.search](../../server/routes/api/documents/documents.ts#L1034-L1120).
Expected findings — **Blocking:** silent behavior change to a **public API default** — every existing API consumer's results change. Needs opt-in, not new default. **Important:** does the share-mode search branch respect the filter (public shares must never surface archived)? **Optional:** UI toggle placement.
Rule: *changing a default is changing the contract.*

## Kata 5: "Refactor: move membership checks into Document model"
Author intent: DRY — moves `includesMembership` logic from the policy file into `Document.canUserAccess(user)`.
Files this resembles: [server/policies/document.ts](../../server/policies/document.ts#L275-L308).
Expected findings — **Blocking:** breaks the architecture's ownership rule (policies own decisions; now two homes) and the client-serialization path (`serialize` iterates registered abilities — a model method is invisible to it, so UI policies drift). **Important:** policies are synchronous-by-design; a model method invites async creep. **Optional:** if duplication was the itch, extract a helper *within* policies.
Lesson: DRY across a **boundary** is usually wrong — duplication is cheaper than coupling.

## Kata 6: "Add index to speed up documents.list"
Author intent: migration adding `CREATE INDEX documents_team_updated ON documents(teamId, updatedAt)`.
Files this resembles: [server/migrations](../../server/migrations) recent index migrations.
Expected findings — **Important:** non-`CONCURRENTLY` index on the largest table = write lock during deploy (boot blocks on pending migrations, so this is downtime); needs `CONCURRENTLY` + the house pattern for it. Where's the `EXPLAIN` evidence this index is what the planner wants (the hot query filters `deletedAt`/`archivedAt` too — partial index better?). **Optional:** name per convention; down migration.
Also **Blocking** if: no `down`.

## Kata 7: "Client: optimistic delete for comments"
Author intent: remove the comment from the store before the API confirms; restore on failure.
Files this resembles: [CommentsStore](../../app/stores/CommentsStore.ts), [Store.remove](../../app/stores/base/Store.ts#L174-L221).
Expected findings — **Important:** restore-on-failure must restore *exactly* (paranoid models set `deletedAt` locally — is un-setting it enough? child comments cascade — restored too?); websocket echo of the real deletion may arrive mid-rollback (idempotent handling?). **Optional:** is optimistic UX even wanted for destructive actions (house pattern check: what do other destructive flows do?).
Lesson: optimistic updates are a distributed-consistency feature wearing a UX costume.

## Kata 8: "Log full event payloads for debugging"
Author intent: adds `Logger.info("worker", JSON.stringify(event))` to the worker loop.
Files this resembles: [worker.ts L45–48](../../server/services/worker.ts#L44-L49).
Expected findings — **Blocking:** events can contain user content and PII (`data` is arbitrary JSONB — [Event.ts L59–65](../../server/models/Event.ts#L59-L65)) → logs become a data-protection liability. Log names/ids (the existing line already does). **Important:** log volume/cost at events-per-edit rates. **Optional:** debug-level + sampling if truly needed.

## Kata 9: "Support markdown export for a single revision"
Author intent: adds `revisions.export` endpoint reusing `DocumentHelper.toMarkdown`.
Files this resembles: [revisions routes](../../server/routes/api/revisions), [presenters/document.ts L52–55](../../server/presenters/document.ts#L52-L55).
Expected findings — **Important:** authorization must be `listRevisions`-equivalent on the *document* (revision policy delegates — check [policies/revision.ts](../../server/policies/revision.ts)); rate limit (conversion is CPU-heavy); content from *old* revisions may reference since-deleted attachments (signed URL behavior?). **Optional:** filename/content-disposition conventions (reuse `contentDisposition` like documents.export does).
This kata rewards knowing that *old data escapes current invariants* — revision content predates whatever rules you added since.

---

## Scoring

Per kata: 2 pts each blocking finding you got, 1 pt each important, −1 for any "blocking" label on a style nit (severity inflation is a real reviewer failure). 18+ across the nine = you review like a solid mid-level. Timed interview versions of katas 1, 3, 5: [08-interview-prep/05](../08-interview-prep/05-debugging-and-code-review-rounds.md).
