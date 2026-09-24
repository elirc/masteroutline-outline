# ADR-007: Yjs CRDT Collaboration with Triple Document Representation

**Status:** Active-with-Scar-Tissue

## Context

Real-time multi-user editing is the product's defining feature. The system
predates its collaborative era: documents were originally markdown `text`,
and an entire ecosystem (search, exports, API consumers) was built on that
column. Retrofitting collaboration required a conflict-resolution strategy, a
server component, and a migration path for years of existing documents —
without a flag-day conversion of a corpus whose members might not be opened
for months.

## Decision

**Conflict resolution:** Yjs CRDTs, served by a Hocuspocus server run as the
`collaboration` service (ADR-001), extended via in-house extensions in
`server/collaboration/` (auth, persistence, logging, metrics, connection
limits).

**Storage:** each document carries **three representations** on one row
(`server/models/Document.ts`): `state` (binary Y.Doc — authoritative),
`content` (Prosemirror JSON projection), `text` (markdown projection).
Projections are refreshed asynchronously (`DocumentUpdateTextTask.ts`) and
are allowed to be briefly stale.

**Migration:** lazy. `PersistenceExtension.onLoadDocument` builds the Y.Doc
from `content`/`text` under a `SELECT ... FOR UPDATE` row lock the first time
a legacy document is opened for editing — converting the corpus one open at a
time, indefinitely.

**Persistence cadence:** debounced. `documentCollaborativeUpdater.ts` writes
the Y.Doc back in a transaction guarded by `SET LOCAL lock_timeout = '15s'`,
recording session collaborators and emitting events (ADR-006).

## Alternatives Considered

1. **Operational Transformation** (Google-Docs style, central sequencer).
   Rejected: OT demands an always-authoritative server ordering every op;
   CRDTs tolerate offline edits and multi-server merge, and mature libraries
   (Yjs) existed with Prosemirror bindings (`y-prosemirror`).
2. **Locking / last-writer-wins** ("document is being edited by X").
   Rejected: it's the anti-feature; the product's demo *is* two cursors in
   one paragraph.
3. **Big-bang offline conversion** of all documents to Yjs state. Rejected:
   a conversion bug on rarely-opened documents would surface months later as
   silent corruption; lazy conversion means a failed conversion is noticed by
   an active user immediately.
4. **Single representation (state only), derive on read.** Rejected:
   projecting markdown/JSON from a CRDT blob on every search-index, export,
   and API read is prohibitive; consumers predating collaboration keep
   working against projections.

## Consequences

**Positive:**
- Offline edits, multi-tab, and concurrent typing merge without a central
  sequencer; client resend-on-reconnect is safe by CRDT math.
- Legacy consumers (search, export, public API) kept working through the
  migration with zero flag day.
- Debounce keeps write amplification sane (multi-MB blobs not written per
  keystroke).

**Negative:**
- **The trap:** every reader must know which representation it reads and how
  stale it may be; "search shows old text for a minute" is accepted anomaly.
  The `withoutState` scope exists because loading blobs accidentally is a
  memory incident.
- Debounced persistence = a durability gap on hard crash, mitigated (not
  closed) by client resend.
- The lock choreography is scar tissue: lazy conversion's `FOR UPDATE` and
  the persister's writes convoy on hot rows — the 15s `lock_timeout` is a
  tombstone of a pool-exhaustion incident.
- Yjs state grows monotonically with edit history; blob size is a permanent
  operational watch-item.
- Collaboration bypasses the HTTP middleware stack entirely, so auth,
  logging, and metrics all needed parallel implementations as Hocuspocus
  extensions (`AuthenticationExtension.ts`, `LoggerExtension.ts`,
  `MetricsExtension.ts`, `ConnectionLimitExtension.ts`).

## Evidence

- `server/collaboration/PersistenceExtension.ts` — lazy conversion under
  lock; the no-lock fast path when `state` exists.
- `server/commands/documentCollaborativeUpdater.ts` — `SET LOCAL
  lock_timeout`, transactional write-back, collaborator attribution.
- `server/models/Document.ts` — `state`/`content`/`text`, `withoutState`.
- `server/queues/tasks/DocumentUpdateTextTask.ts` — projection refresh.
- `shared/editor/` — the schema both sides must share (ADR-002 is a
  precondition of this ADR).
