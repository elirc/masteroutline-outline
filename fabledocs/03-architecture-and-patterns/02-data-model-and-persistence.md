# Data Model and Persistence

## The entity graph (core)

```
Team 1─* User        Team 1─* Collection 1─* Document (self-ref parentDocumentId)
User *─* Group (GroupUser)
Document 1─* Revision, 1─* Comment (self-ref parentCommentId), 1─* View, 1─* Share
Access grants: UserMembership / GroupMembership → (collection | document) with permission level
Everything ──▶ Event (audit)     Async jobs w/ state ──▶ FileOperation, Import/ImportTask
```

Verified from [server/models/index.ts](../../server/models/index.ts) and the model files; 295 migrations in [server/migrations](../../server/migrations) from 2016 to present — a decade of schema evolution you can read like sediment layers.

Key modeling decisions to study (each is an interview story):

1. **Multi-tenancy by column.** `teamId` on nearly everything; **isolation** is enforced per-query (e.g. [documents.list's where](../../server/routes/api/documents/documents.ts#L119-L132)), not by schema/database separation. Tradeoff: cheap ops, but one forgotten filter = cross-tenant leak; mitigations are scopes + policy checks + tests.
2. **Document content trinity.** `title` (string column), `content` (ProseMirror JSON), `state` (Yjs binary). The `withState`/`withoutState` scopes ([Document.ts L129–148](../../server/models/Document.ts#L129-L148)) exist because `state` is heavy — loaded only when a route needs it (`includeState: true` in [documents.update L1264–1268](../../server/routes/api/documents/documents.ts#L1264-L1268), for collab-aware edits). *Invariant to defend:* `content` and `state` describe the same document; the collaborative pipeline is responsible for keeping them converged.
3. **Lifecycle columns over status enums.** `publishedAt`, `archivedAt`, `deletedAt` timestamps encode state machine + audit-when in one field. Derived predicates (`isDraft`, `isActive`) live on models and are used by policies ([document policy](../../server/policies/document.ts#L86-L102)).
4. **Soft delete (paranoid)** with scheduled purges — see [Pattern 11](05-pattern-catalog.md).
5. **Denormalizations, each with a maintenance cost:** `collection.documentStructure` (sidebar tree; maintained by commands under transaction), `document.collaboratorIds`, popularity/insights rollups (maintained by tasks like [UpdateDocumentsPopularityScoreTask](../../server/queues/tasks/UpdateDocumentsPopularityScoreTask.ts) and [RollupDocumentInsightsTask](../../server/queues/tasks/RollupDocumentInsightsTask.ts)). Rule of thumb worth quoting: *a denormalization is a cache; every cache needs an owner and an invalidation story.*

## Transactions and consistency

- **Boundary:** the [transaction middleware](../../server/middlewares/transaction.ts) wraps each mutating route; commit = middleware return, rollback = any throw. Everything inside must pass `transaction` explicitly — the repo's #1 latent-bug shape is a query inside a wrapped handler that omits it (it runs auto-committed outside the txn).
- **Events are transactional; queue publish is post-commit** ([Event.ts L82–97](../../server/models/Event.ts#L82-L97)) — the consistency hinge, discussed in [04-side-effects…](04-side-effects-async-and-reliability.md).
- **Locking is exceptional and deliberate:** the only explicit row lock found in the main flows is comment anchoring ([comments.create](../../server/routes/api/comments/comments.ts#L28)) protecting the merged Yjs artifact. Everything else accepts last-write-wins. **Consistency** language for interviews: per-request writes are strongly consistent (single txn); cross-service effects (revisions, search, notifications) are *eventually consistent by design*.
- Read patterns: no read replicas assumed; heavy reads are shaped by scopes and pagination ([paginateQuery's count-skipping](../../server/routes/api/middlewares/pagination.ts#L21-L47)).

## How to change the schema safely here

1. `yarn db:create-migration --name=add-x-to-y` → edit the generated file in [server/migrations](../../server/migrations) (both `up` and `down`).
2. Model change in [server/models](../../server/models) (+ typed column decorators); presenter change *only if* the field should reach clients; client model change if consumed.
3. **Expand → migrate → contract** for anything not brand-new: add nullable column → backfill (a task, not the migration, if the table is big) → make required/remove old. Never rename in one step: deploys overlap versions, and pending migrations *block boot* ([checkPendingMigrations](../../server/index.ts#L51-L60)), so a slow migration is an outage.
4. Indexes: add `CONCURRENTLY` for big tables (check recent migration files for house style — e.g. [20260621000000-add-search-queries-user-created-index.js](../../server/migrations/20260621000000-add-search-queries-user-created-index.js)).
5. **Rollback** plan = your `down` actually works + the previous app version tolerates the new schema (that's what expand/contract buys). "What's your rollback?" is a standard interview follow-up — this is the answer's shape.

## What juniors miss vs seniors check

- Junior: adds the column, forgets the presenter is the exposure gate (either leaks the field or wonders why the client can't see it).
- Junior: writes a `down` that drops a column that `up` backfilled — destroying data on rollback without a warning comment.
- Senior: asks "who else writes this table?" (commands, tasks, collab persistence) before adding an invariant.
- Senior: checks paranoid + unique interactions (soft-deleted row still holds the unique slot?).
- Senior: sizes the migration against production row counts and lock behavior before merging.

## Interview angle

1. *"Design the schema for nested documents"* → parent pointer + denormalized structure tree, with the sync cost stated ([system design Step 3](../08-interview-prep/04-system-design-from-this-repo.md)).
2. *"How do you do zero-downtime migrations?"* → expand/migrate/contract + boot-blocking check as the forcing function.
3. *"When do you use a transaction / a lock?"* → middleware default for mutations; locks only where concurrent writers corrupt merged state (comment anchoring example — memorize it, it's vivid and rare).

## Drill

Design (on paper, migration + model + presenter deltas) the change: "documents get a `wordCount` column, shown in the UI, kept accurate." Decide: computed where (on save? in [DocumentUpdateTextTask](../../server/queues/tasks/DocumentUpdateTextTask.ts)-style async task?), backfilled how, exposed where. Self-grade — Basic: valid migration. Solid: async backfill + presenter gate + client model. Strong: names the collaborative-editing write path as the sneaky second writer that must also update it, and chooses the event-driven task for exactly that reason.
