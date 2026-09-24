# ADR-005: Sequelize with Decorators; Explicit Transaction-per-Mutation

**Status:** Active-Eroding

## Context

Postgres is the sole database (multi-tenancy via `teamId` columns, not
schemas). Mutations are multi-row by nature — moving a document repositions
siblings; updating one writes the row *and* an `events` row that every
downstream consumer (websockets, webhooks, revisions, search) depends on. A
mutation that half-commits starves those consumers silently. The team needed
(a) an ORM strategy and (b) a transaction discipline that survives dozens of
contributors.

## Decision

**ORM:** `sequelize-typescript` — decorator-annotated model classes in
`server/models/` over shared bases (`server/models/base/IdModel.ts`, paranoid
variants), with validators as decorators and named scopes (e.g.
`Document.unscoped()`, the `withoutState` scope that avoids accidentally
loading multi-megabyte CRDT blobs). Schema changes are explicit migrations in
`server/migrations/` only — no sync.

**Transactions:** a middleware (`server/middlewares/transaction.ts`) wraps
every mutating route in `sequelize.transaction()` and exposes the handle as
`ctx.state.transaction`. All business choreography lives in **commands**
(`server/commands/documentMover.ts`, `userProvisioner.ts`, ...) that accept
the transaction explicitly and pass it to every query and to `Event.create`.
Propagation is explicit-by-parameter, deliberately not ambient.

## Alternatives Considered

1. **Query builder / raw SQL (Knex-style).** Rejected: the model layer isn't
   just queries — validation decorators, lifecycle hooks feeding the event
   queue (`server/models/Event.ts`), scopes, and paranoid deletes would all
   be hand-rolled.
2. **Ad hoc transactions opened inside commands.** This was the observed
   starting point; rejected after recurring "document row updated, event row
   didn't" partial commits.
3. **AsyncLocalStorage/CLS implicit transaction propagation** (Sequelize
   supports it). Rejected: when async context is lost (stray timer, callback
   boundary), CLS fails *silently into autocommit* — the worst failure mode
   for a correctness mechanism. Explicit passing fails visibly in review
   instead.

## Consequences

**Positive:**
- Mutation + event row commit atomically; the event spine (ADR-006) can trust
  that an event implies its mutation.
- Commands are independently testable units with explicit dependencies
  (`documentMover.test.ts`).
- Scopes like `withoutState` encode performance safety into the data layer.

**Negative:**
- `transaction` is the most-passed parameter in the codebase; every command
  signature carries it, and **forgetting it compiles and passes low-
  concurrency tests**. Enforcement is convention + review, not types — this
  is why the status is Eroding: nothing structural stops a new autocommit
  query from slipping in.
- Sequelize's typing requires ceremony (`InferAttributes`, the `@Fix`
  decorator in `server/models/decorators/` patching class identity).
- ORM abstraction leaks at the hard edges: `documentCollaborativeUpdater.ts`
  drops to raw SQL for `SET LOCAL lock_timeout = '15s'`, and
  `PersistenceExtension.ts` reaches for `transaction.LOCK.UPDATE` — the
  concurrency-critical paths are written *around* the ORM, not with it.

## Evidence

- `server/middlewares/transaction.ts` — the whole policy in ~20 lines.
- `server/commands/documentMover.ts` — transaction threading through a
  multi-row choreography.
- `server/models/base/`, `server/models/decorators/Fix.ts`,
  `server/models/Document.ts` (scopes) — the ORM strategy.
- `server/commands/documentCollaborativeUpdater.ts` — raw `SET LOCAL`, the
  escape hatch.
- `server/migrations/` — migration-only schema evolution.
