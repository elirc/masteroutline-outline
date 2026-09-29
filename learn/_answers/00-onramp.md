# SEALED — answers to learn/00-onramp.md

Only open this after you have written your own answer. Every answer below is reasoned from source at `37ca0d5`.
Nothing was executed (no `node_modules`). Where library behaviour matters, it is marked **UNVERIFIED** with the
command that confirms it.

---

## E1
1. `"q3-plan-AbC123"`. The pathname is `/doc/q3-plan-AbC123/edit`, so the split is `["", "doc", "q3-plan-AbC123", "edit"]` and the element after `"doc"` is the slug.
2. `"q3-plan-AbC123"`. The leading `/` gets `sharedEnv.URL` prepended, and `URL.pathname` drops the `#heading`.
3. **`""`** (empty string). `indexOf("doc")` is `-1`, so the code reads `split[0]`. That is `""`, because every pathname starts with `/`. `?? undefined` does **not** turn `""` into `undefined`, since `??` only catches `null` and `undefined`.
4. **`undefined`**. The split is `["", "doc"]` and `split[2]` doesn't exist.
5. **`undefined`**. `new URL("not a url")` throws, and the `catch` returns early.

The surprise is **#3**. Code that checks `!== undefined` would go on to look up a document with id `""`. Callers
that use a truthiness check (`if (slug)`) are safe.

**Grading:** 5/5 is required to count this one. Most people get #3 wrong.

## E2
Reference cases, added inside the existing `describe` in `shared/utils/parseDocumentSlug.test.ts`:

```ts
describe("current behaviour on non-document urls (characterization)", () => {
  // PasteHandler.tsx:131 guards with isDocumentUrl() and `if (slug)`, so "" is harmless there.
  it("returns an empty string when there is no /doc/ segment", () => {
    expect(parseDocumentSlug("https://app.outline.dev/collection/eng-xyz")).toBe("");
  });
  it("returns undefined when /doc has no slug after it", () => {
    expect(parseDocumentSlug("https://app.outline.dev/doc")).toBeUndefined();
  });
  it("returns undefined for an unparseable url", () => {
    expect(parseDocumentSlug("not a url")).toBeUndefined();
  });
});
```

Rubric:
- Uses `toBe("")` and `toBeUndefined()`, **not** `toBeFalsy()`. A falsy check would hide exactly the difference you are pinning.
- The function itself is unchanged.
- The test is in the same file, not a new directory.
- The guard comment names a real caller: `app/editor/extensions/PasteHandler.tsx:131` (`isDocumentUrl` + `if (slug)`), or `server/routes/api/urls/urls.ts:62` (it passes the result straight into `loadPublicShare` as `documentId`, so find out what `loadPublicShare` does with `""`; that makes a good follow-up question).
- **UNVERIFIED:** run `yarn install && yarn test shared/utils/parseDocumentSlug.test.ts`.

## E3
Order:
1. `router.post("documents.info", ...)` at `server/routes/api/documents/documents.ts:573`.
2. `auth({ optional: true })` from `server/middlewares/authentication.ts`. Because it is optional, a missing token does **not** throw here.
3. `validate(T.DocumentsInfoSchema)`: the body must have `id` or `shareId`.
4. The handler: `getAPIVersion`, then `getTeamFromContext`. There is no `shareId`, so it takes the `else` branch.
5. `documentLoader({ id, user })` in `server/commands/documentLoader.ts`, which calls `Document.findByPk(id, { paranoid: false, userId })`.
6. `authorize(...)` from `server/policies/cancan.ts:182`.
7. `presentDocument` (`server/presenters/document.ts`), then `presentPolicies`, then `ctx.body = { data, policies }`. With `apiVersion >= 2` the payload is wrapped in `{ document }`.

Failures:
- a. `AuthenticationError("Authentication required")` at `documents.ts:630`, in the handler, **not** the middleware. This is 401.
- b. `authorize(user, "restore", document)` in `documentLoader.ts` throws `AuthorizationError` (`cancan.ts:189`). That maps to **403** (`server/errors.ts:32`). If no row matches at all, you get `NotFoundError` (404).
- c. `authorize(user, "read", document)` fails the same way, with 403.

Bonus insight: a missing id gives 404, but an existing doc you can't read gives 403. That is an existence
signal. It matters little here because ids are UUIDs, but it is exactly the kind of thing to raise in a review.
`isTrialImport` gives 402 (`PaymentRequiredError`).

## E4
1. The order is: `Redis` (only when `env.REDIS_COLLABORATION_URL` is set), `Throttle`, `ConnectionLimitExtension`, `EditorVersionExtension`, `AuthenticationExtension`, `PersistenceExtension`, `APIUpdateExtension`, `ViewsExtension`, `LoggerExtension`, `MetricsExtension`.
2. Cheap rejection comes first. A flood of connections or an over-limit client gets turned away before any JWT or DB work happens, and an unauthenticated socket never reaches `PersistenceExtension.onLoadDocument`, which reads (and may lock) the document row. Putting persistence before auth would give anonymous users a way to make the server load, and lazily convert, any document id.
3. It counts **store attempts**. It increments on every `onStoreDocument`. That includes the "no changes" early return in `PersistenceExtension` (lines 125-128), the `isUnchanged` no-op in the updater, and the swallowed failure (lines 138-143). That is the observability gap behind upstream candidate C. **UNVERIFIED:** whether Metrics' hook would still run if an earlier extension's hook *threw*, which depends on how `@hocuspocus/server` chains hooks. Today `PersistenceExtension` swallows every error, so the question doesn't come up. It matters for candidate C, though (see learn/01 D2).

## E5
| Column | Written by | When | Suppressed? |
|---|---|---|---|
| `state` | (1) `PersistenceExtension.ts:85-94`, the lazy conversion | first collaborative load of a doc whose `state` is null, inside a `LOCK.UPDATE` transaction | `silent: true, hooks: false`: no `updatedAt` bump, no hooks |
| `state` | (2) `documentCollaborativeUpdater.ts:99-113` | the debounced save (3 s idle / 10 s max, or last disconnect) | `hooks: false` only, so `updatedAt` **does** change |
| `content` | `documentCollaborativeUpdater.ts:99-113` (plus the non-collaborative API update path, out of scope) | same save as (2), canonicalized via `Node.fromJSON(...).toJSON()` (lines 59-62) | `hooks: false` |
| `text` | `DocumentUpdateTextTask.ts:16,30` | asynchronously, after `RevisionsProcessor` decides to make a revision | `silent: true` only. **Hooks still run.** `silent` in Sequelize suppresses the timestamp, not the hooks |

The key point for grading: `text` is **not** written during the collaborative save, so it can be stale for
minutes. That is ADR-007's "projections may be briefly stale", and it is the root of fire drill 5.

## E6
- a. It depends on the Redis set, not on the viewer. If `collaborators:<id>` is empty (for example, `RevisionsProcessor.ts:28` already `del`-ed it after the last revision), line 125 returns with nothing written and no transaction opened. If an earlier editor's set hasn't been cleared yet, the viewer's disconnect still triggers the full updater.
- b. **Not empty.** Only `RevisionsProcessor` deletes the key (line 28), and only for `done: true` or `.debounced` events. So `documentCollaborativeUpdater` runs: it opens a transaction, sets `lock_timeout`, takes `LOCK.UPDATE` on the row, and canonicalizes. Then `isEqual(document.content, content)` (line 63) holds and it returns at line 70-72. **Nothing is written, but a row lock was still taken.** Seeing that cost is the real point of the exercise.
- c. Guard 1 (an empty set) is free, with no DB access. It answers "did anyone act?" but can be stale. Guard 2 (`isEqual`) is authoritative on "did the content change?". It also absorbs y-prosemirror's key-order and empty-`attrs` noise, but it costs a locked read.

## E7
An `events` row would have given you: (1) an **audit trail** (who changed what and when, visible to admins and
the API); (2) **durability** (the row survives a Redis/Bull outage, and you could in principle replay it; see
fire drill 3 and ADR-006); (3) visibility to retention reasoning such as `CleanupOldEventsTask`. The trade:
collaborative saves fire every few seconds per active document, so persisting each one would flood the table,
for events whose only consumers are internal processors.

## E8
- a. `findByPk` returns null, `!document?.content` is true, so it returns. No save.
- b. Same early return. No save. This is the untested branch (upstream candidate B).
- c. `text` is set to the markdown serialization. `franc(...)` on fewer than 50 characters returns `"und"`, and `iso6393To1["und"]` is `undefined`, so `document.language = undefined`. `save({ silent: true })` **is** called. **UNVERIFIED** (needs deps): `yarn install`, then check `node_modules/iso-639-3` and `franc`'s `minLength` behaviour. It is also worth pinning what Sequelize writes for an `undefined` attribute.

Scheduling: `RevisionsProcessor.ts:46`. It is skipped when (i) the event is `documents.update` with `done` falsy
(line 21), or (ii) `content` **and** `title` equal the latest revision's (lines 38-44). In case (ii), `text`
isn't refreshed either. That's fine only because nothing changed since that revision.

## E9
1. The delayed jobs: from t=0, running at t=5. From t=2, running at t=7. From t=6, running at t=11.
2. Line 40: `if (document.updatedAt > new Date(event.createdAt)) return;`
   - The t=5 job: `updatedAt` is t=2, later than `createdAt` t=0, so it **aborts**.
   - The t=7 job: `updatedAt` is t=6, later than t=2, so it **aborts**.
   - The t=11 job: `updatedAt` is about t=6, **not** later than `createdAt` t=6, so it **re-emits** `documents.update.debounced`.
3. The revision happens at about **t=11** (one revision for the whole burst), unless the last client disconnected earlier. That save carries `done: true`, and `RevisionsProcessor` then acts on the plain `documents.update` right away.
4. `updatedAt` is set by Sequelize when `document.update(...)` runs (`hooks: false` does not stop timestamps; only `silent` would). `Event.schedule` computes `createdAt: now` **afterwards** (`documentCollaborativeUpdater.ts:99` runs before line 115). So `updatedAt <= createdAt`, and the self-comparison never aborts. If the order were reversed, every debounce would abort itself and revisions would only come from `done: true`. That's a plausible, subtle regression. (Reasoned from code order. Not run.)
