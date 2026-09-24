# Writing Tests Here: Seven Recipes

Concrete recipes using this repo's actual helpers. Commands are **inferred** from scripts (`yarn test <path>` runs a single file — [AGENTS.md](../../AGENTS.md#L153-L168)); test DB setup one-time via `make test` or the sequelize commands in the [Makefile](../../Makefile).

Common ingredients: factories ([server/test/factories.ts](../../server/test/factories.ts)), [getTestServer / withAPIContext / mockTaskSchedule](../../server/test/support.ts), msw for outbound HTTP ([server/test/msw.ts](../../server/test/msw.ts)).

All snippets below are **illustrative fake code: not from this repo** — they follow the shapes in real neighboring tests, which are linked as the ground truth.

## Recipe 1: Happy-path route test

Ground truth style: [documents.test.ts](../../server/routes/api/documents/documents.test.ts#L1-L50).

```ts
// Illustrative fake code: not from this repo
const server = getTestServer();

it("updates the title", async () => {
  const user = await buildUser();
  const document = await buildDocument({ teamId: user.teamId, userId: user.id });
  const res = await server.post("/api/documents.update", {
    body: { token: user.getJwtToken(), id: document.id, title: "New" },
  });
  const body = await res.json();
  expect(res.status).toEqual(200);
  expect(body.data.title).toEqual("New");
});
```

Note the auth transport: route tests pass `token` in the body (a supported transport — [authentication.ts L105–113](../../server/middlewares/authentication.ts#L105-L113)); check the real file for the current helper idiom (TestServer also accepts a user object — [TestServer.ts](../../server/test/TestServer.ts#L14-L40)).

## Recipe 2: Validation failure

Assert both status *and* message shape, because the message format is a contract of [validate.ts](../../server/middlewares/validate.ts#L17-L25):

```ts
// Illustrative fake code: not from this repo
const res = await server.post("/api/documents.update", {
  body: { token: user.getJwtToken(), id: document.id, title: 42 },
});
expect(res.status).toEqual(400);
expect((await res.json()).message).toContain("title");
```

## Recipe 3: Permission failure (the one that matters)

Topology-building is the skill. Cover: outsider (other team), viewer role, guest, revoked membership.

```ts
// Illustrative fake code: not from this repo
it("blocks users from another team", async () => {
  const outsider = await buildUser();               // own team auto-built
  const document = await buildDocument();           // different team
  const res = await server.post("/api/documents.update", {
    body: { token: outsider.getJwtToken(), id: document.id, title: "X" },
  });
  expect(res.status).toEqual(403);                  // check the real convention: 403 vs 404 here!
});
```

The 403-vs-404 assertion is worth verifying against an existing cross-team test — the scoped `findByPk` may yield authorization-on-null (→ 403 from `authorize(user, action, null)`) — knowing which is *the* contract detail.

## Recipe 4: Cross-tenant isolation on list endpoints

The nastiest leaks are in lists/search, not single reads:

```ts
// Illustrative fake code: not from this repo
it("never returns another team's documents", async () => {
  const user = await buildUser();
  await buildDocument();                            // other team's doc
  const res = await server.post("/api/documents.list", {
    body: { token: user.getJwtToken() },
  });
  const body = await res.json();
  expect(body.data.map((d) => d.teamId ?? "absent")).not.toContain(/* other team */);
  // Better: assert exact ids returned === ids you created for this user.
});
```

Rule from [testing strategy](01-testing-strategy.md): assert on *your* created data exactly; presence-of-foreign checks are the one exception where you assert absence.

## Recipe 5: Async side effect (event → task scheduled)

Direct processor test, ground truth: [RevisionsProcessor.test.ts](../../server/queues/processors/RevisionsProcessor.test.ts).

```ts
// Illustrative fake code: not from this repo
it("creates a revision when the edit session ends", async () => {
  const document = await buildDocument();
  const processor = new RevisionsProcessor();
  await processor.perform({
    name: "documents.update",
    documentId: document.id,
    actorId: document.createdById,
    teamId: document.teamId,
    data: { done: true },
  } as DocumentEvent);
  expect(await Revision.count({ where: { documentId: document.id } })).toEqual(1);
  // Idempotency assertion — run it twice:
  await processor.perform(/* same event */);
  expect(await Revision.count({ where: { documentId: document.id } })).toEqual(1);
});
```

The run-it-twice assertion is the habit this curriculum most wants to install.

## Recipe 6: Task scheduling without executing

Use [mockTaskSchedule](../../server/test/support.ts#L43) to assert "X was enqueued" without running the worker:

```ts
// Illustrative fake code: not from this repo
const schedule = mockTaskSchedule();
// ...trigger the flow...
expect(schedule).toHaveBeenCalledWith(expect.objectContaining({ documentId: doc.id }));
```

(Check the helper's actual return/assert shape in support.ts before using — it exists precisely so tests don't need Redis.)

## Recipe 7: Client store test (jsdom project)

Ground truth style: existing store/model tests under app/ (e.g. [app/stores/CollectionsStore.ts's neighbors](../../app/models/Collection.test.ts)).

```ts
// Illustrative fake code: not from this repo
it("merges websocket partials into existing models", () => {
  const store = new RootStore();
  const doc = store.documents.add({ id: "d1", title: "Old" });
  store.documents.add({ id: "d1", title: "New" });          // partial, like a ws event
  expect(store.documents.get("d1")).toBe(doc);              // identity preserved!
  expect(doc.title).toEqual("New");
});
```

The `toBe` (identity) assertion tests the *reason* the merge exists ([Store.add](../../app/stores/base/Store.ts#L152-L172)).

---

## Running them

```bash
yarn test server/routes/api/documents/documents.test.ts    # single file (preferred)
yarn test:server                                            # whole backend project (slow — avoid)
yarn test:app                                               # jsdom project
```

Checklist before opening a PR with tests: test names state behavior not implementation ("blocks users from another team", not "returns 403"); every new mutation has at least the recipe-3 negative; async logic has a run-twice assertion; no sleeps — await the actual promise.
