# System Design From This Repo: "Design a Team Wiki (Notion/Confluence-lite)"

Reverse-engineering Outline into a whiteboard exercise. You will likely never be asked "design Outline" — you *will* be asked "design a collaborative docs product", "design commenting", or "design public sharing", and having actually read an implementation puts you in the top decile of candidates. Practice this as a 35–45 minute monologue with interruptions.

Cross-reference: [03-architecture-and-patterns/06-architecture-critique.md](../03-architecture-and-patterns/06-architecture-critique.md) is the "what would you change" half of this exercise.

## Step 1 — Requirements (3–5 min)

Functional: multi-tenant teams; documents in collections; rich-text collaborative editing; permissions (team/collection/doc, groups, guests); comments + mentions; notifications (in-app + email); full-text search; public share links; version history; import/export; integrations.

Non-functional: real-time feel (<100ms echo for collaborators), read-heavy (wikis are ~90/10 read/write), documents are small (KBs) but numerous, correctness of *permissions* matters more than write throughput, self-hostable (constrains infra choices — one of Outline's real constraints: Postgres + Redis + S3 only).

- **Junior answer sounds like:** jumps straight to tables and endpoints.
- **Mid-level adds:** read/write ratio, permission-correctness as the top risk, real-time as a separate subsystem.
- **Senior includes:** the self-hosting constraint shaping every later choice ("no DynamoDB/Kafka — the deployment story is part of the product").

## Step 2 — API sketch (5 min)

Say: "I'd use an RPC-style JSON API — `documents.update`, `documents.move` — because wikis have many operations that map badly to REST verbs." That is literally Outline's choice ([documents.ts](../../server/routes/api/documents/documents.ts), ~30 verbs on one noun). Sketch the uniform pipeline: authenticate → validate (Zod) → rate-limit (per-cost strategy) → transaction → authorize → mutate → present. Anchors: [validate.ts](../../server/middlewares/validate.ts), [transaction.ts](../../server/middlewares/transaction.ts), [rateLimiter usage](../../server/routes/api/documents/documents.ts#L1034-L1040).

Stronger/simpler alternative to mention: plain REST + PATCH works fine if the operation set is small; GraphQL buys client-shaped queries at the cost of authz complexity per field.

## Step 3 — Data model (8 min)

Draw: `teams ⟵ users`, `collections(teamId)`, `documents(teamId, collectionId, parentDocumentId, createdById, urlId, content jsonb, state bytea, publishedAt, archivedAt, deletedAt)`, `revisions`, `shares`, `comments`, `user_memberships` / `group_memberships` (polymorphic-ish grants), `events` (audit), `stars/pins/subscriptions/notifications`, `attachments`, `search: tsvector on documents`.

Points that score:
- **Tenant column everywhere** + every query filtered by it (show you know isolation is per-query discipline: [documents.list starts with `teamId`](../../server/routes/api/documents/documents.ts#L119-L132)).
- **Document tree:** parent pointer (`parentDocumentId`) *plus* a denormalized ordered tree per collection (`collection.documentStructure` JSON) for cheap sidebar rendering — Outline's actual denormalization, with its real cost: structure updates must stay in sync with document rows (this is why [documentMover](../../server/commands/documentMover.ts) is a command with a transaction).
- **Content storage:** ProseMirror JSON (`content`) for rendering/search + Yjs binary (`state`) for collaboration. Two representations of one truth = an invariant to defend; say who writes which and when.
- **Soft delete** (`deletedAt`) + purge task; **drafts** = `publishedAt IS NULL`, not a separate table.
- Ordering via **fractional indexes** (stars/pins/structure), not integer positions.

Junior: one `documents` table with `text`. Mid: memberships, soft delete, revisions as separate table, indexes on `(teamId, updatedAt)` etc. Senior: dual content representations and their sync risk; audit events table doubling as an outbox-ish bus source; retention policy for revisions/events.

## Step 4 — Permissions (6 min) — the differentiating section

Model: role (admin/member/viewer/guest) × grants (collection default access, per-collection membership, per-document membership, group memberships) → effective ability. Implement as a **policy engine**: declarative rules composing (`update` ⊃ `read`), evaluated server-side, and **serialized to the client** so the UI renders the same decisions ([cancan.ts serialize](../../server/policies/cancan.ts#L127-L155), [document policy](../../server/policies/document.ts)). Note the performance trick: preload membership data via query scopes so policy evaluation is synchronous ([withMembership scope](../../server/models/Document.ts#L179-L228)).

Follow-up you should expect: "how do you keep list endpoints fast under permissions?" Answer: compute the user's accessible collection ids once (`user.collectionIds()` in [documents.list](../../server/routes/api/documents/documents.ts#L165-L171)) and filter *in SQL*, never post-filter in app code (breaks pagination, leaks counts).

## Step 5 — Real-time (8 min)

Two channels, deliberately separate (Outline's actual split):
1. **Entity updates** — socket.io rooms per team/collection/document; server-side mutations emit events post-commit; a processor fans them to rooms; clients merge into a normalized cache. Anchors: [Event afterCommit](../../server/models/Event.ts#L82-L97) → [worker fan-out](../../server/services/worker.ts#L53-L88) → [WebsocketProvider](../../app/components/WebsocketProvider.tsx#L253-L262).
2. **Collaborative text** — CRDT (Yjs) over a dedicated websocket service (Hocuspocus), persisting snapshots ([server/collaboration/PersistenceExtension.ts](../../server/collaboration/PersistenceExtension.ts)). Explain *why CRDT over OT*: no central transform sequencing, offline merge, library maturity.

Senior points: the websockets queue is consumed **only** by the websockets service ([worker.ts L63–66](../../server/services/worker.ts#L63-L66)) because only those processes hold the socket connections (via Redis adapter). Deploys must tolerate old clients receiving new event payloads. Revision history is built by **debouncing** the edit-event firehose ([RevisionsProcessor](../../server/queues/processors/RevisionsProcessor.ts)) rather than snapshotting per keystroke.

## Step 6 — Async pipeline (5 min)

Everything not needed for the response goes through events → queues: notifications → emails, webhooks, search indexing, backlinks, insights. Reference architecture: per-consumer jobs with independent retries ([Pattern 9](../03-architecture-and-patterns/05-pattern-catalog.md)); idempotent consumers because Bull retries; priorities so emails beat housekeeping ([BaseTask options](../../server/queues/tasks/base/BaseTask.ts#L48-L60)).

Name the gap like a senior: post-commit enqueue can drop events on crash — a true **outbox relay** over the existing `events` table would close it. Saying "here's the gap and the fix, and why it's probably fine at this scale" is exactly the calibrated-risk talk interviews reward.

## Step 7 — Scaling walk (5 min)

- Reads: Postgres with good indexes goes very far for wiki-sized data; add read replicas before caches.
- The **hot spot** is actually websocket fan-out and collaboration sessions → scale those horizontally (Redis adapter/dedicated collab Redis — Outline encodes this constraint at [server/index.ts L39–48](../../server/index.ts#L39-L48)).
- Search: Postgres FTS first, provider seam for Elasticsearch later ([SearchProviderManager](../../server/utils/SearchProviderManager.ts), [SearchIndexProcessor](../../server/queues/processors/SearchIndexProcessor.ts) already emitting the events you'd need).
- Workers scale by queue depth; queues already partitioned (events/processors/tasks/websockets).

## Variation prompts (practice each as a 5-minute delta)

1. **"Now make share links support search and nested trees."** — Outline did: share tree fencing, `includeChildDocuments`, publicness-by-`cannot` ([Flow 2](../01-codebase-cartography/05-key-flows.md)).
2. **"Now add 10x traffic."** — Which service melts first and why? (websockets fan-out; then search; DB last.)
3. **"Now guarantee webhook delivery."** — outbox + delivery table with retry state ([plugins/webhooks](../../plugins/webhooks/server) already models deliveries).
4. **"Now add offline editing."** — CRDT already buys most of it; discuss IndexedDB persistence (y-indexeddb is in [package.json](../../package.json)) and auth token expiry offline.
5. **"Now multi-region."** — the honest answer: CRDT sync multi-master is fine, Postgres isn't; region-pin teams (tenancy makes this tractable).

## Grading yourself

Record a full run. Check: did you state the read/write ratio? Did permissions get ≥5 minutes? Did you name one denormalization *and its maintenance cost*? Did you volunteer one failure mode unprompted (outbox gap, dual content representations)? Did every major choice come with a rejected alternative? That last habit — "we could do X, I'd choose Y because Z" — is the single strongest mid-to-senior signal.
