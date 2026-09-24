# Fake-Code Contrasts

Ten bad-vs-better pairs. **Every snippet here is illustrative fake code — none of it is from this repo** — but each "better" shape mirrors a real pattern here (anchored). The bad versions are the instincts a junior brings from tutorials; the point is to recognize them in your own diffs.

---

## 1. Post-filtering permissions vs filtering in the query

```ts
// Illustrative fake code: not from this repo — BAD
const docs = await Document.findAll({ where: { teamId } });
const visible = docs.filter((d) => userCanSee(user, d)).slice(0, 25);
```
```ts
// Illustrative fake code: not from this repo — BETTER
const collectionIds = await user.collectionIds();
const docs = await Document.findAll({
  where: { teamId, collectionId: collectionIds }, limit: 25, offset,
});
```
Why: post-filtering loads the whole table, breaks pagination (page 1 may filter to 3 items), and leaks counts. Real pattern: [documents.list](../../server/routes/api/documents/documents.ts#L165-L171). Failure it prevents: quadratic slowdowns and permission-shaped pagination bugs.

## 2. Trusting the client's teamId vs deriving from the session

```ts
// Illustrative fake code: not from this repo — BAD
const { teamId, documentId } = ctx.request.body;
await Document.update({ archivedAt: new Date() }, { where: { id: documentId, teamId } });
```
```ts
// Illustrative fake code: not from this repo — BETTER
const { user } = ctx.state.auth;                       // identity from token
const document = await Document.findByPk(documentId, { userId: user.id });
authorize(user, "archive", document);                  // grant, not just tenancy
await document.archiveWithCtx(ctx);
```
Why: tenancy from the request body is attacker-controlled (IDOR). Real pattern: every handler takes `user` from `ctx.state.auth` and authorizes the loaded object ([documents.update](../../server/routes/api/documents/documents.ts#L1261-L1270)).

## 3. Returning ORM objects vs presenting

```ts
// Illustrative fake code: not from this repo — BAD
ctx.body = { data: await User.findByPk(id) };  // ships passwordDigest, tokens, everything
```
```ts
// Illustrative fake code: not from this repo — BETTER
ctx.body = { data: presentUser(user) };        // explicit allowlist of fields
```
Why: serialization by default is disclosure by default. Real pattern: [server/presenters](../../server/presenters/document.ts) — and note presenters *subtract more* for public viewers (L97–129).

## 4. Side effects before commit vs after

```ts
// Illustrative fake code: not from this repo — BAD
await sequelize.transaction(async (t) => {
  const doc = await Document.create(data, { transaction: t });
  await emailQueue.add({ docId: doc.id });   // queued; txn may still roll back!
});
```
```ts
// Illustrative fake code: not from this repo — BETTER
await sequelize.transaction(async (t) => {
  const doc = await Document.create(data, { transaction: t });
  t.afterCommit(() => void emailQueue.add({ docId: doc.id }));
});
```
Why: the worker can race ahead of (or outlive) a rolled-back transaction — emails about documents that don't exist. Real pattern: [Event.ts L82–97](../../server/models/Event.ts#L82-L97).

## 5. Query-per-item vs batch (N+1)

```ts
// Illustrative fake code: not from this repo — BAD
for (const doc of documents) {
  doc.source = await FileOperation.findByPk(doc.importId);  // N queries
}
```
```ts
// Illustrative fake code: not from this repo — BETTER
const sources = await FileOperation.findAll({ where: { id: { [Op.in]: importIds } } });
const byId = new Map(sources.map((s) => [s.id, s]));
for (const doc of documents) { doc.source = byId.get(doc.importId) ?? null; }
```
Why: N+1s are invisible at dev scale, dominant at prod scale. The better version is nearly verbatim the real fix in [presentDocuments L147–176](../../server/presenters/document.ts#L147-L176).

## 6. UI recomputing permissions vs rendering server policy

```tsx
// Illustrative fake code: not from this repo — BAD
{(user.isAdmin || doc.createdById === user.id || membership?.canEdit) && <EditButton />}
```
```tsx
// Illustrative fake code: not from this repo — BETTER
const can = usePolicy(doc.id);
{can.update && <EditButton />}
```
Why: the bad version *will* drift from the server's rules (it already misses group grants and team mutability). Real pattern: serialized policies ([Pattern 14](../03-architecture-and-patterns/05-pattern-catalog.md)).

## 7. Swallowed errors vs typed failure

```ts
// Illustrative fake code: not from this repo — BAD
try { await webhook.deliver(payload); } catch (e) { console.log("webhook failed"); }
```
```ts
// Illustrative fake code: not from this repo — BETTER
try { await webhook.deliver(payload); }
catch (err) {
  Logger.error("Webhook delivery failed", toError(err), { webhookId: webhook.id });
  throw err;  // let the queue retry with backoff; onFailed handles exhaustion
}
```
Why: catching to log-and-continue converts a retryable failure into silent data loss; in queue context, *rethrowing is the recovery mechanism*. Real pattern: [worker.ts L140–155](../../server/services/worker.ts#L140-L155).

## 8. Stale-closure polling vs subscription

```tsx
// Illustrative fake code: not from this repo — BAD
useEffect(() => {
  const i = setInterval(() => setTitle(document.title), 1000); // `document` frozen at mount
  return () => clearInterval(i);
}, []);
```
```tsx
// Illustrative fake code: not from this repo — BETTER
const Title = observer(() => <h1>{document.title}</h1>);  // MobX tracks the dereference
```
Why: the effect closes over the mount-time `document`; and polling for push-available data wastes everything. Real pattern: observer components over the store cache ([framework mental models](../02-stack-and-language-mastery/02-framework-mental-models.md)).

## 9. Integer positions vs fractional index

```ts
// Illustrative fake code: not from this repo — BAD
await Star.update({ position: Sequelize.literal("position + 1") }, { where: { position: { [Op.gte]: target } } });
await star.update({ position: target });   // renumber the world on every drag
```
```ts
// Illustrative fake code: not from this repo — BETTER
star.index = fractionalIndex(prevStar?.index ?? null, nextStar?.index ?? null);
await star.save();
```
Why: renumbering is O(n) writes and a lock party under concurrency. Real pattern: fractional-index usage ([Pattern 12](../03-architecture-and-patterns/05-pattern-catalog.md)).

## 10. Casually widening a public contract

```ts
// Illustrative fake code: not from this repo — BAD (in a presenter)
return { ...document.toJSON(), internalFlags: document.flags };  // spread = every future column ships
```
```ts
// Illustrative fake code: not from this repo — BETTER
return { id: document.id, title: document.title, /* each field a decision */ };
```
Why: spreading a model into a response makes future schema changes *silently public* — contract change by accident. Real pattern: presenters enumerate fields explicitly ([document.ts L57–91](../../server/presenters/document.ts#L57-L91)).

---

Drill: for each pair, name the *category* from memory a day later (permissions-in-query, tenancy-from-token, presentation-allowlist, post-commit-effects, batching, policy-driven-UI, retry-via-rethrow, subscription-over-polling, fractional-ordering, explicit-contracts). These ten categories cover a remarkable fraction of real review findings — they're also the backbone of [04-review-katas.md](04-review-katas.md).
