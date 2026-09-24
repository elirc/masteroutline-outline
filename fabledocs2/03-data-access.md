# 03 — Data Access: Sequelize, Transactions Everywhere, and a Document With Three Bodies

## What it is and why it exists

Postgres is the only database. Access goes through `sequelize-typescript` —
decorator-annotated model classes in `server/models/` extending shared bases
in `server/models/base/` (`IdModel`, paranoid variants). Migrations live in
`server/migrations/` and run via `yarn db:migrate`; there is no auto-sync,
ever. Multi-tenancy is a `teamId` column on essentially every table, enforced
by scopes and by the policy layer — not by separate schemas or databases,
because self-hosters typically have exactly one team and cloud needs cheap
tenant creation.

Business mutations don't live in models or routes; they live in **commands**
(`server/commands/` — `documentMover.ts`, `documentDuplicator.ts`,
`userProvisioner.ts`...). A command takes models + a transaction, does the
multi-step work, writes the `Event` row, and returns. Routes stay thin;
models stay declarative; commands hold the choreography.

## The hard decisions

### 1. Transaction-per-mutation as middleware

`server/middlewares/transaction.ts` wraps mutating routes in
`sequelize.transaction()` and parks the handle on `ctx.state.transaction`.
Every DB call in the request must pass it explicitly.

Alternatives we rejected: (a) no ambient transactions, open them ad hoc in
commands — this is how we started, and we kept finding endpoints where the
document row updated but the event row didn't, which silently starved every
event consumer (no websocket update, no revision, no webhook); (b) CLS/
AsyncLocalStorage-based implicit transaction propagation, which Sequelize
supports — magical, and when the magic fails (a stray `setImmediate`, a
library that loses async context) it fails *silently* into autocommit, which
is the worst possible failure mode for a correctness feature.

So we chose explicit-but-mandatory. The tradeoff that bothers me daily:
`transaction` is now the most-passed parameter in the codebase, it's on
practically every function signature in `server/commands/`, and forgetting to
pass it compiles fine and works fine in tests with low concurrency. The type
system can't save you. Code review and a couple of lint conventions are the
actual enforcement, and I don't love that.

### 2. The document's three representations

Look at `server/models/Document.ts`: a document has `state` (the binary Yjs
CRDT blob — the *actual* source of truth), `content` (Prosemirror JSON), and
`text` (markdown). Three representations of the same content, on one row.

Why? History and consumers. `text` predates collaborative editing and is what
search indexing, exports, and half the public API were built on. `content`
exists because rendering markdown loses structure the editor needs. `state`
arrived with Yjs and *must* be authoritative because CRDT merge history can't
be reconstructed from a snapshot. The migration path lives in
`server/collaboration/PersistenceExtension.ts#onLoadDocument`: when a doc
with no `state` is opened for editing, we build the Y.Doc from
`content`/`text` under a `SELECT ... FOR UPDATE` lock — lazily converting the
corpus one open at a time, years into the migration.

The rejected alternative was a big-bang migration converting every document's
markdown to Yjs state offline. We ran the math on cloud data volume and on
the risk of a conversion bug corrupting documents nobody would open (and thus
notice) for months, and chose lazy conversion. Right call, but the cost is
permanent: every consumer must ask "which representation am I reading, and
how stale is it?" `DocumentUpdateTextTask.ts` exists solely to refresh the
projections from state, and "search shows the old text for a minute" is a
known, accepted anomaly. This is the single biggest conceptual trap in the
codebase. Respect it.

### 3. Writing collaborative state: locks and debounce

`server/commands/documentCollaborativeUpdater.ts` persists the in-memory
Y.Doc back to Postgres. Read it carefully: it opens a transaction, issues
`SET LOCAL lock_timeout = '15s'`, and re-reads the row before writing. That
lock timeout is a scar, not a flourish (see the incident below). Persistence
is debounced by Hocuspocus — we don't write on every keystroke — and revision
snapshots are debounced again on top (`RevisionsProcessor.ts` via
`DebounceProcessor.ts`), because "one revision per keystroke" and "one
revision per editing session" differ by four orders of magnitude in storage.

What still bothers me: debounce means a hard process crash can lose the last
few seconds of unpersisted Y.Doc state. Clients usually resend from their own
copy on reconnect — the CRDT makes that safe — but "usually" depends on a
client still being alive to reconnect. We've accepted a small durability gap
in exchange for not writing multi-megabyte blobs on every keypress. Postgres
`LISTEN/NOTIFY`-based streaming or a WAL-style update log were both sketched
and shelved as complexity we couldn't afford.

## Evidence to go read

- `server/models/Document.ts` — the three columns, the scopes (note
  `withoutState` — loading state blobs by accident is a memory incident).
- `server/models/base/` and `server/models/decorators/` — what every model
  inherits; the `@Fix` decorator has a story, ask me sometime.
- `server/commands/documentMover.ts` + its test — the best example of a
  multi-row command that would be incorrect without a transaction.
- `server/collaboration/PersistenceExtension.ts` — lazy state creation under
  lock.
- `server/migrations/` — skim the last dozen; you learn schema history faster
  here than anywhere.

## An incident this design caused

The `lock_timeout` in `documentCollaborativeUpdater.ts` postdates a bad week.
A team pasted an enormous document (embedded base64 images in the state
blob), and persisting it held a row lock long enough that the *lazy state
conversion* path in `PersistenceExtension` — which takes `FOR UPDATE` on the
same row — queued behind it. Connections piled up, the pool drained, and
unrelated API requests started timing out. Root cause chain: big blob → long
write → lock convoy → pool exhaustion. Fixes: the 15-second local lock
timeout (fail one document's save instead of the server), attachment-based
image handling to keep blobs sane, and pool-utilization metrics. When you see
a bare `SET LOCAL` in a codebase, assume it's a tombstone.

## Before you move on

1. A document's `text` says one thing and its `state` says another. Which is
   right, why can this happen legitimately, and what task reconciles them?
2. Why did we choose explicit transaction-passing over AsyncLocalStorage
   propagation, and what's the failure mode of each when they go wrong?
3. You're writing a command that moves a document and updates ten siblings'
   positions. Walk through what happens with and without the shared
   transaction when two users do it concurrently.
