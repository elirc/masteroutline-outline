# Annotation Drills

Ten real excerpts. For each: annotate **inputs, outputs, dependencies, invariants, side effects, failure modes** *before* reading the answer key hints. Do them on paper or in a scratch file — writing is the drill. All excerpts are real code (trimmed with `…`), verified at commit `8e032f8cd`.

**Self-grading rubric (every drill):**
- **Basic (1pt each):** correct inputs/outputs.
- **Solid (2pts):** + at least one true invariant and one side effect, precisely stated ("mutates ctx.input", not "does validation stuff").
- **Strong (3pts):** + a failure mode that isn't in the code's happy path, and what you'd test.
Target: 24/30 across the ten.

---

## Drill 1 — [server/middlewares/transaction.ts#L13-L20](../../server/middlewares/transaction.ts#L13-L20)

```ts
export function transaction() {
  return async function transactionMiddleware(ctx: AppContext, next: Next) {
    await sequelize.transaction(async (t: Transaction) => {
      ctx.state.transaction = t;
      return next();
    });
  };
}
```

Annotate, then check: What commits the transaction? What rolls it back? What is the invariant every *downstream* database call must maintain? (Hint: passing `transaction` explicitly — a query that omits it runs *outside* the txn and can see/write inconsistent state. That's the classic bug this pattern invites.) Failure mode worth naming: the handler awaits an external HTTP call while holding an open transaction → connection pool exhaustion under load.

## Drill 2 — [server/middlewares/validate.ts#L11-L27](../../server/middlewares/validate.ts#L11-L27)

```ts
return async function validateMiddleware(ctx: APIContext, next: Next) {
  try {
    ctx.input = { ...ctx.input, ...schema.parse(ctx.request) };
  } catch (err) {
    if (err instanceof ZodError) {
      const { path, message } = err.issues[0];
      …
      throw ValidationError(errMessage);
    }
    ctx.throw(toError(err));
  }
  return next();
};
```

Check: why parse `ctx.request` and not `ctx.request.body`? (Schemas can validate `body`, `query`, or both — look at any schema file.) Side effect: mutates `ctx.input` by *merging* — what earlier middleware could have written there, and what does merge order imply? Failure mode: only the first Zod issue is surfaced (see Ticket 1).

## Drill 3 — [server/models/Event.ts#L82-L97](../../server/models/Event.ts#L82-L97)

```ts
@AfterSave
static async enqueue(model: Event, options: SaveOptions<InferAttributes<Event>>) {
  if (options.transaction) {
    (options.transaction.parent || options.transaction).afterCommit(
      () => void globalEventQueue().add(model)
    );
    return;
  }
  void globalEventQueue().add(model);
}
```

Check: why `transaction.parent || transaction`? (Nested transactions from `findOrCreate` — the comment above the excerpt cites the Sequelize issue; the `afterCommit` must attach to the *outermost* txn or it never fires.) What does `void` discard, and what's the consequence? Invariant: no consumer ever sees an event for uncommitted data. Failure mode: crash after COMMIT before `add` → lost event; enqueue rejection → silently dropped (no retry, no log at this site).

## Drill 4 — [server/policies/cancan.ts#L99-L119](../../server/policies/cancan.ts#L99-L119)

```ts
for (const ability of matchingAbilities) {
  if (!ability.condition) { continue; }
  const result = ability.condition(performer, target, options);
  if (!result || seenConditions.has(result)) { continue; }
  seenConditions.add(result);
  if (typeof result === "string") { membershipIds.push(result); }
  else { hasNonMembershipMatch = true; }
}
return membershipIds.length > 0 ? membershipIds : hasNonMembershipMatch;
```

Check: the return type is `boolean | string[]` — what do the strings represent, and who consumes them? (Membership ids explaining *why* access was granted; see `includesMembership` returning ids at [document.ts L292–307](../../server/policies/document.ts#L275-L308).) Invariant: conditions must be pure and synchronous. Failure mode: a condition that queries the DB would need to be async — the design *forces* preloading, which is the point.

## Drill 5 — [server/commands/documentUpdater.ts#L100-L125](../../server/commands/documentUpdater.ts#L100-L125)

```ts
const changed = document.changed();
…
if (publish && cId) {
  …
  await document.publish(ctx, { collectionId: cId, data: eventData });
} else if (changed) {
  document.lastModifiedById = user.id;
  document.updatedBy = user;
  await document.saveWithCtx(ctx, undefined, { data: eventData });
} else if (done) {
  await Event.schedule({ ...event, actorId: user.id, teamId: document.teamId });
}
```

Check: three mutually exclusive paths — name each and what it persists. Why does the `done`-only path use `Event.schedule` (no audit row) instead of `Event.create`? What updates `updatedAt` and thus client "edited X ago" displays? Failure mode: a caller setting both `publish` and unchanged content — which branch wins and is the event emitted once?

## Drill 6 — [app/stores/base/Store.ts#L152-L172](../../app/stores/base/Store.ts#L152-L172)

```ts
@action
add = (item: PartialExcept<T, "id"> | T): T => {
  const ModelClass = this.model;
  if (!(item instanceof ModelClass)) {
    const existingModel = this.data.get(item.id);
    if (existingModel) {
      existingModel.updateData(item);
      return existingModel;
    }
    …
    const newModel = new ModelClass(item, this);
    this.data.set(newModel.id, newModel);
    return newModel;
  }
  this.data.set(item.id, item);
  return item;
};
```

Check: why *merge into the existing instance* instead of replacing it? (Object identity — MobX observers and React components hold references; replacement would orphan them.) Inputs arrive from three sources — name them (fetch, mutation response, websocket). Failure mode: a partial from a websocket clobbering newer local fields — what ordering assumption is being made?

## Drill 7 — [server/routes/api/middlewares/pagination.ts#L21-L47](../../server/routes/api/middlewares/pagination.ts#L21-L47) (`paginateQuery`)

```ts
const isClientRequest = !!ctx.headers["x-client-version"];
if (isClientRequest) {
  const results = await find({ offset, limit: limit + 1 });
  const hasNextPage = results.length > limit;
  if (hasNextPage) { results.pop(); }
  return { results, pagination: { …, total: hasNextPage ? offset + limit + 1 : offset + results.length } };
}
```

Check: what does fetching `limit + 1` buy? (Skips a COUNT query — the web app only needs "is there a next page".) Why keep exact COUNT for non-client (API) callers? Invariant: `total` for client requests is a *lower bound*, not a count — what UI could this break? This is a great example of a **contract** subtly differing by caller.

## Drill 8 — [server/queues/processors/RevisionsProcessor.ts#L25-L44](../../server/queues/processors/RevisionsProcessor.ts#L25-L44)

```ts
const key = Document.getCollaboratorKey(event.documentId);
const collaboratorIds = await Redis.defaultClient.smembers(key);
await Redis.defaultClient.del(key);
const document = await Document.findByPk(event.documentId, { paranoid: false, rejectOnEmpty: true });
const previous = await Revision.findLatest(document.id);
if (previous && isEqual(document.content, previous.content) && document.title === previous.title) {
  return;
}
```

Check: this runs with Bull retries — walk each line asking "safe to run twice?" Where is the idempotency guard? What's lost if the process dies between `smembers` and the revision insert? Why `paranoid: false`? (Revisions for docs deleted mid-flight.)

## Drill 9 — [server/routes/api/comments/comments.ts, inside `comments.create`](../../server/routes/api/comments/comments.ts#L28)

```ts
if (anchorText) {
  // Acquire the row lock on the document directly when anchoring so a
  // concurrent inline comment can't overwrite our state update.
  await Document.unscoped().findOne({
    where: { id: documentId },
    attributes: ["id"],
    transaction,
    lock: transaction.LOCK.UPDATE,
  });
}
```

Check: why `unscoped()` and `attributes: ["id"]`? (Cheapest possible lock acquisition — no scopes, no columns.) What exact race does this prevent (two commenters anchoring simultaneously; both read doc `state`, both write back — last write erases the first mark)? Why does the lock come *before* the scoped `findByPk`? Failure mode: lock ordering vs other flows that lock documents — deadlock potential (investigate: does any other flow lock documents?).

## Drill 10 — [server/services/worker.ts#L62-L88](../../server/services/worker.ts#L62-L88)

```ts
try {
  if (name === "WebsocketsProcessor") {
    await websocketQueue().add(job.data);
  } else if (ProcessorClass.applicableEvents.includes(event.name) || ProcessorClass.applicableEvents.includes("*")) {
    if (!ProcessorClass.shouldQueue || (await ProcessorClass.shouldQueue(event))) {
      await processorEventQueue().add({ event, name });
    }
  }
} catch (error) {
  … err = error;
}
…
if (err) { throw err; }
```

Check: why catch per-processor but rethrow at the end? (One bad processor mustn't block others' jobs *this pass*, but the global job must still retry.) Follow-up: on retry, processors that already got jobs get them **again** — which downstream property does that demand? (Consumer idempotency.) Why is WebsocketsProcessor special-cased by *name*?

---

After all ten: pick the two excerpts you scored lowest on and write a 5-line explanation of each as if teaching a new hire. Teaching-quality explanation = you actually own it. These ten excerpts also seed the timed reviews in [08-interview-prep/05-debugging-and-code-review-rounds.md](../08-interview-prep/05-debugging-and-code-review-rounds.md).
