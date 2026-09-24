# The Collaborative Editing Save Path, File by File

This trace goes one level below `fabledocs3/adr-007-crdt-triple-representation.md`
(the ADR states the decision and its evidence list; this page walks the
control flow those files implement, call by call, keystroke to event). Read
the ADR first if you haven't — this assumes ADR-007's vocabulary (triple
representation, lazy conversion, debounced persistence).

Every step below is a real file:line. Verified against baseline commit
`37ca0d5` (this repo's `training/` baseline — see `training/README.md`).
Line numbers drift; symbol names are the fallback.

## 0. Client connects

`app/scenes/Document/components/MultiplayerEditor.tsx:94-98` constructs a
`HocuspocusProvider` pointed at `${env.COLLABORATION_URL}/collaboration`
(a websocket). The extension that wires this into ProseMirror is
`app/editor/extensions/Multiplayer.ts`.

## 1. Server accepts the socket

`server/services/collaboration.ts:24-71` is the service entrypoint (only
runs in a process started with `--services=collaboration`, or in dev's
combined process — see `server/index.ts` service flags, ADR-001). It:
- upgrades the raw `http.Server` connection on `path = "/collaboration"`
  (`collaboration.ts:73-138`), pulling `documentId` off the URL;
- hands the socket to a Hocuspocus `Server` instance configured at
  `collaboration.ts:44-71` with `debounce: 3000, timeout: 30000,
  maxDebounce: 10000` and an **ordered** extension list. Order matters:
  `ConnectionLimitExtension` and `AuthenticationExtension` run before
  `PersistenceExtension`, so an unauthenticated or over-limit connection
  never reaches the DB.

## 2. Document load (first open in this server process)

`server/collaboration/PersistenceExtension.ts#onLoadDocument` (lines 20-97):
1. If the in-memory Y.Doc Hocuspocus is about to hand back already has
   content, bail (line 29) — don't reload from Postgres on every reconnect.
2. Unlocked read: `Document.unscoped().findOne({ attributes: ["state"] })`
   (line 34). If `state` (binary Y.Doc blob, the authoritative column per
   ADR-007) is already populated, decode it with `Y.applyUpdate` and return
   — the common case, no transaction needed.
3. **Lazy conversion path** (state is null — a pre-collaboration or
   never-opened-since-migration document): open a `sequelize.transaction`
   and re-read **with `lock: transaction.LOCK.UPDATE`** (line 55) — this is
   the row lock ADR-007 calls out as scar tissue. Double-check `state`
   again inside the lock (another process may have converted it first,
   line 64) before building a Y.Doc from `content` (ProseMirror JSON) or
   falling back to `text` (markdown) via `ProsemirrorHelper.toYDoc`
   (lines 71-83), then writing the derived `state` back with
   `{ silent: true, hooks: false }` (line 85-94) — silent/no-hooks because
   this is an internal projection write, not a user edit; it must not fire
   the `AfterUpdate` hook chain (see step 5's warning) or re-emit events.

## 3. Live edits: presence, not persistence

`PersistenceExtension#onChange` (lines 99-111) fires on every Yjs update
broadcast (i.e., on essentially every keystroke across all connected
clients) but **does not touch Postgres**. It only records the editing
user's id into a Redis set keyed by `Document.getCollaboratorKey(documentId)`
(`Redis.defaultClient.sadd`). This is how the system later attributes a
save to "who typed since the last save" without touching the DB on every
keystroke — the debounce in step 1's Hocuspocus config is what turns a
stream of `onChange` calls into an infrequent `onStoreDocument`.

## 4. Debounced flush: `onStoreDocument`

`PersistenceExtension#onStoreDocument` (lines 113-144) fires after
Hocuspocus's debounce window (3s idle, 10s max) or on last-client-disconnect.
It reads back the Redis collaborator set (line 124); **if empty, it returns
without writing anything** (line 125-128, logged as "No changes") — this is
the mechanism that stops a save from happening when nobody actually edited
(e.g., a viewer who just opened and closed the doc). Otherwise it calls
`documentCollaborativeUpdater` (`server/commands/documentCollaborativeUpdater.ts`)
inside a `try/catch` that only logs on failure (lines 138-143) — see the
observability gap flagged in `training/navigation/02-upstream-contribution-kit.md`
candidate 3.

## 5. The actual write: `documentCollaborativeUpdater`

`server/commands/documentCollaborativeUpdater.ts` (the whole file is short —
read it directly):
1. Opens `sequelize.transaction` and immediately runs
   `SET LOCAL lock_timeout = '15s'` (lines 34-37) — the tombstone ADR-007
   names as evidence of a past pool-exhaustion incident from this lock
   convoying with the load-path's `LOCK.UPDATE` in step 2.
2. Re-reads the document `.scope("withoutState")` **with `LOCK.UPDATE`**
   (lines 39-52) — `withoutState` exists so this hot path never pulls the
   multi-MB Yjs blob into memory just to lock the row (ADR-007's "memory
   incident" note).
3. Derives the two projections from the live Y.Doc: `state` via
   `Y.encodeStateAsUpdate(ydoc)` (line 54) and `content` via
   `yDocToProsemirrorJSON` **round-tripped through
   `Node.fromJSON(schema, ...).toJSON()`** (lines 59-62) — the comment here
   is load-bearing: the round-trip exists purely to canonicalize JSON key
   order/empty-attrs so an unrelated diff tool (or a revision comparison)
   doesn't see spurious no-op edits.
4. `isUnchanged = isEqual(document.content, content)` (line 63): if the
   canonicalized content didn't actually change, **return without writing**
   (line 70-72) — a second no-op guard, this time on content rather than on
   "did anyone edit."
5. Attribution: `lastModifiedById` prefers the last id in
   `sessionCollaboratorIds` (the Redis set from step 3), falling back to the
   document's existing value if the doc is soft-deleted (lines 64-68).
   Collaborator ids are merged from three sources — existing
   `document.collaboratorIds`, this session's Redis set, and Yjs's own
   `Y.PermanentUserData` client list (lines 79-86) — because a client can
   edit and disconnect before the socket-level Redis bookkeeping in step 3
   ever ran for it.
6. `document.update({ content, state, lastModifiedById, collaboratorIds,
   editorVersion }, { transaction, hooks: false })` (lines 99-113) — **note
   `text` (the markdown projection) is not written here.** ADR-007 calls
   this out: `text` is refreshed asynchronously, not synchronously with the
   collaborative save.
7. Still inside the same transaction, `Event.schedule({ name:
   "documents.update", ... data: { multiplayer: true, done: isLastConnection
   } })` (lines 115-127). `Event.schedule` (`server/models/Event.ts:140-148`)
   pushes straight onto the Bull `processorEvents` queue — **it does not
   write an `events` row**, unlike `Event.create`/`Event.createFromContext`
   used by the plain HTTP mutation path. Read the doc comment above it
   (lines 136-139): this is a deliberate low-durability shortcut for
   high-frequency multiplayer saves, at the cost of these saves being
   invisible to the audit trail and to `CleanupOldEventsTask`-style
   reasoning (ADR-006). `done: isLastConnection` matters downstream — see
   step 7.

## 6. Fan-out: who reacts to `documents.update`

`server/queues/processors/DebounceProcessor.ts` and
`server/queues/processors/RevisionsProcessor.ts` both list `documents.update`
in `applicableEvents` — Bull dispatches the same event to every processor
that declares it (see `server/queues/index.ts` for the registration).

- **`DebounceProcessor`** (lines 7-54) implements a *distributed* debounce
  layered on top of Hocuspocus's in-process one: on `documents.update` it
  re-enqueues itself as `documents.update.delayed` after 5 minutes
  (0.5 min in dev, line 21), then on firing checks whether
  `document.updatedAt` has moved past the event's `createdAt` (line 40) —
  if another save landed in the meantime, it drops itself silently. Only
  when nothing newer has landed does it re-emit as
  `documents.update.debounced`. This exists so a document being edited
  continuously for an hour gets ~1 revision, not one per 3-second Hocuspocus
  flush.
- **`RevisionsProcessor`** listens for `documents.publish`,
  `documents.update`, and `documents.update.debounced`. For the plain
  `documents.update` case it **only proceeds if `event.data?.done` is true**
  (line 21) — i.e., a mid-session multiplayer save (`done: false` from step
  5) is ignored here; only the final save when the last client disconnects,
  or the distributed-debounced event, triggers revision/text work. It then
  schedules `DocumentUpdateTextTask` (line 46) — see step 7 — and (reading
  further into the file, not reproduced here) calls `revisionCreator` to
  snapshot a `Revision` row.

## 7. Text projection refresh

`server/queues/tasks/DocumentUpdateTextTask.ts` (33 lines, the whole file):
loads the document, serializes its `content` JSON back to markdown with
`serializer.serialize(node)` (line 16) and writes `document.text`, plus
runs language detection (`franc`, lazily imported to keep it off the
startup path — lines 18-23) into `document.language`. It saves with
`{ silent: true }` (line 30) — a Sequelize hook-suppression convention you
should now recognize from steps 2 and 5. **This is the "projections are
allowed to be briefly stale" clause of ADR-007 made concrete**: from the
moment `onStoreDocument` commits (step 5) to the moment this task runs, any
reader of `document.text` (search indexing, markdown export, the public
API) sees the pre-edit markdown.

## The whole path, condensed

```
keystroke → Yjs update (client)
  → websocket → Hocuspocus.onChange → Redis SADD collaborator (no DB write)
  → [3s idle / 10s max debounce]
  → Hocuspocus.onStoreDocument → documentCollaborativeUpdater
        (txn, LOCK.UPDATE, SET LOCAL lock_timeout)
        → Document.update({ state, content, ... })   [text NOT written here]
        → Event.schedule("documents.update", { multiplayer:true, done })
  → Bull "processorEvents" queue, fanned out to:
        DebounceProcessor  (5 min distributed debounce → documents.update.debounced)
        RevisionsProcessor (only if done:true or .debounced)
              → DocumentUpdateTextTask (async: content → text, language)
              → revisionCreator (snapshot Revision row)
```

## Where staleness is observable (try this)

1. Two browser tabs, same doc, same team. Edit in tab A, watch tab B's
   cursor — that's steps 0-3, no Postgres involved.
2. Query `documents.text` for that row (e.g. via `yarn sequelize` console
   or a debugger breakpoint in `DocumentUpdateTextTask.perform`) within the
   3-10s debounce window after editing, then again a minute later. The
   first read is stale; `state`/`content` are already current.
3. Grep `withoutState` in `server/models/Document.ts` — that scope is the
   guardrail against accidentally loading the blob this whole path is built
   to avoid loading on the hot path.

## Interview angle

"Walk me through what happens when I type a character in a shared doc" is a
standard system-design follow-up for this kind of product. The answer that
signals mid/senior level names: (a) why persistence is decoupled from the
CRDT broadcast (steps 0-3 vs 4-5), (b) why there are *two* debounce layers
at different timescales solving different problems (Hocuspocus's 3s memory
buffer vs `DebounceProcessor`'s 5-minute revision-collapse), (c) why
`Event.schedule` here skips the durable `events` table that the HTTP
mutation path uses, and what that trades away (ADR-006), and (d) the two
independent "don't write if nothing changed" guards (empty collaborator set;
`isEqual` content check) and why both are needed (one is about *who* acted,
the other about *what* changed).
