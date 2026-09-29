# 00 — On-ramp: nine warm-ups in Outline's own code

Do these before `training/navigation/02` or `fabledocs4`. Each one takes 10–25 minutes. Write your answer
down **before** you open `learn/_answers/00-onramp.md`. Line numbers match the baseline `37ca0d5`. If they have
drifted, search for the symbol name.

None of these need a running stack. E2 needs `yarn install` if you want to actually run the test.

---

## E1 — Predict the output (shared code, 10 min)

Read `shared/utils/parseDocumentSlug.ts` (25 lines). Assume `sharedEnv.URL = "https://app.outline.dev"` (the
test file sets exactly that). Predict the **exact** return value for each call. Be careful to tell `undefined`
apart from `""`.

1. `parseDocumentSlug("https://app.outline.dev/doc/q3-plan-AbC123/edit")`
2. `parseDocumentSlug("/doc/q3-plan-AbC123#heading")`
3. `parseDocumentSlug("https://app.outline.dev/collection/eng-xyz")`
4. `parseDocumentSlug("https://app.outline.dev/doc")`
5. `parseDocumentSlug("not a url")`

Then answer this: which of 3/4/5 would surprise a caller who writes `if (slug !== undefined)`?

## E2 — Write one characterization test (20 min)

`shared/utils/parseDocumentSlug.test.ts` only tests happy paths: every input contains `/doc/<slug>`. Add a
`describe` block that pins the **current** behaviour for inputs 3, 4 and 5 from E1. The rules:

- Don't change `parseDocumentSlug.ts`. A characterization test records what the code does, not what you wish
  it did ([curriculum 04](../../../opusorganize/apprenticeship/curriculum/04-characterization-tests.md)).
- Put the test in the existing collocated file. `AGENTS.md` says "Do not create new test directories".
- In a comment, name one real caller that is protected by its own guard, and say what that guard is. Find the
  callers with `grep -rn "parseDocumentSlug(" app server`.

**[needs `yarn install`]** To run it: `yarn test shared/utils/parseDocumentSlug.test.ts`.

## E3 — Trace a request end to end (25 min)

Follow `POST /api/documents.info` with body `{ id }` (no `shareId`) from
`server/routes/api/documents/documents.ts:573` to the response body. List every function in order, with
file:line. Then say **where** and **which error** each of these produces:

a. No session cookie or token.
b. The document is soft-deleted, and the user is not allowed to restore it.
c. The document exists, but the user has no read permission.

Hint: `server/commands/documentLoader.ts` and `authorize` from `server/policies`.

## E4 — Read a config and reason about order (15 min)

`server/services/collaboration.ts:44-70` passes Hocuspocus an **ordered** list of extensions.

1. Write out the order. Which one is only present sometimes, and what decides that?
2. Why is it important that `Throttle` and `ConnectionLimitExtension` come before `AuthenticationExtension`, and
   that all three come before `PersistenceExtension`?
3. `MetricsExtension` is last. Look at its `onStoreDocument` (`server/collaboration/MetricsExtension.ts:62`).
   What does `collaboration.change` actually count: saves, attempts, or edits?

## E5 — Explain the schema: one document, three bodies (20 min)

A `documents` row carries the same text three times: `state` (a Yjs binary), `content` (ProseMirror JSON), and
`text` (markdown). Fill in this table from the code, not from the ADR:

| Column | Written by (file:line) | When | Hooks/timestamps suppressed? |
|---|---|---|---|
| `state` | ? (two places) | ? | ? |
| `content` | ? | ? | ? |
| `text` | ? | ? | ? |

Start from `PersistenceExtension.ts`, `documentCollaborativeUpdater.ts` and `DocumentUpdateTextTask.ts`.

## E6 — Predict the write (collaboration, 20 min)

Use `PersistenceExtension.onStoreDocument` (lines 113-144), `documentCollaborativeUpdater.ts`, and
`RevisionsProcessor.ts:25-28`.

a. A viewer opens a document that nobody has edited since the last revision, and closes it without typing.
   `onStoreDocument` fires. Is anything written to Postgres? Which line decides that?
b. User A types one word and the debounced save persists it. A then leaves the tab open without typing, and
   Hocuspocus fires `onStoreDocument` again, for example when another client disconnects. Is the Redis
   collaborator set empty now? (Who deletes `collaborators:<id>`? Search for `getCollaboratorKey`.) Which guard
   stops a second, identical write?
c. Why does the code need **both** guards?

## E7 — Same event name, two durability levels (10 min)

Read the comment above `Event.schedule` in `server/models/Event.ts` (~line 136). The collaborative save calls
`Event.schedule({ name: "documents.update", ... })` (`documentCollaborativeUpdater.ts:115`). Name two things
that an `events` table row would have given you and `schedule` doesn't. Name one reason the authors accepted
that trade.

## E8 — Predict the side effects of a 33-line task (15 min)

`server/queues/tasks/DocumentUpdateTextTask.ts`. For each input, say what happens to `document.text` and
`document.language`, and whether `save` is called:

a. `event.documentId` points to a row that no longer exists.
b. The row exists, but `content` is `null` (for example, a draft that has never been saved collaboratively).
c. `content` holds one short paragraph of about 20 characters. (Look at `minLength: 50` on line 26, and think
   about what `franc` returns when it can't decide.)

Also answer: which processor schedules this task, and under what condition does it **skip** scheduling it even
though the document changed? (`RevisionsProcessor.ts:36-46`.)

## E9 — A timeline puzzle: the distributed debounce (20 min)

This runs in production (`env.isProduction`, so the delay is 5 minutes). Collaborative saves that change content
land at t = 0, t = 2 min and t = 6 min, and each one calls `Event.schedule("documents.update")`. Using
`server/queues/processors/DebounceProcessor.ts`:

1. List every `documents.update.delayed` job and the time it runs.
2. For each one, does it abort or re-emit `documents.update.debounced`? Quote the comparison on line 40.
3. Roughly when does `RevisionsProcessor` create a revision for this burst, given that every mid-session save
   has `done: false`?
4. Subtle point: for the t = 6 min save, is `document.updatedAt` earlier or later than the event's `createdAt`?
   Look at the order of `document.update(...)` and `Event.schedule(...)` inside the transaction.
