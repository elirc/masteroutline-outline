# API and Data-Modeling Question Cards

Twelve cards. These rounds decide mid-level offers more than any other technical round — permissions, pagination, migrations, and transactions are where "built CRUD apps" and "understands systems" diverge.

## Q1: "Design the API for moving a document to another collection."
Round: API/data. Repo anchor: [documents.move](../../server/routes/api/documents/documents.ts#L1354-L1400) + [documentMover](../../server/commands/documentMover.ts).
Junior: `PATCH /documents/:id {collectionId}`. Mid: names the hidden work — permission on *both* target collection and document, child subtree handling, ordering index in the target, transactionality across document rows + both collections' structure trees. Senior: RPC verb honesty (`documents.move` because REST hides the multi-object semantics), the denormalized-tree maintenance cost, and the websocket fan-out to both collections' rooms.

## Q2: "How do you paginate correctly?"
Round: API/data. Repo anchor: [pagination middleware + paginateQuery](../../server/routes/api/middlewares/pagination.ts#L21-L47).
Junior: offset/limit. Mid: the count-cost problem and this repo's fetch-limit+1 trick for "has next page"; caps on limit; why permission filters must be *inside* the query. Senior: offset instability under concurrent writes (page drift), when cursor/keyset pagination wins, and the honest note that `total` here is a lower bound for client requests — a documented contract nuance.

## Q3: "Validation — where and how many times?"
Round: API/data. Repo anchor: the [gauntlet table](../03-architecture-and-patterns/03-validation-auth-and-permissions.md); Zod at the edge ([documents/schema.ts](../../server/routes/api/documents/schema.ts)), semantic checks in handlers/commands, DB constraints last.
Junior: "validate in the controller." Mid: three layers with different jobs (shape / state-dependent rules / invariant backstop) + types generated from schemas. Senior: the duplication question — client-side validation mirrors *rules* not code; and what happens at version skew (old client, new schema — [debugging Scenario 5](../05-quality-engineering/03-systematic-debugging.md)).

## Q4: "Model permissions for teams, groups, and per-document sharing."
Round: API/data. Repo anchor: memberships (user↔{collection,document}, group↔{collection,document}) + roles + policies ([03-architecture/03](../03-architecture-and-patterns/03-validation-auth-and-permissions.md)).
Junior: role column. Mid: grant tables + precedence rules + the effective-permission computation; guest/viewer role floors. Senior: the *evaluation architecture* — preload grants via scopes, decide synchronously in policies, serialize decisions to the client ([Patterns 4/5/14](../03-architecture-and-patterns/05-pattern-catalog.md)); performance of list endpoints under permissions (compute accessible-collection ids once).

## Q5: "Walk me through a zero-downtime schema change."
Round: API/data. Repo anchor: [data-model page's expand/migrate/contract](../03-architecture-and-patterns/02-data-model-and-persistence.md); boot-gating migrations ([server/index.ts L51–60](../../server/index.ts#L51-L60)); `CONCURRENTLY` indexes.
Junior: "run the migration." Mid: additive-first sequences, backfill via tasks not migrations for big tables, both app versions tolerating both schemas. Senior: the [K-6 kata](../06-contribution-practice/04-refactor-and-design-katas.md)'s live example (`data`→`changes` columns mid-flight in this repo — [Event.ts L59–73](../../server/models/Event.ts#L59-L73)) and rollback as a first-class design input.

## Q6: "When do you use transactions, and what are the traps?"
Round: API/data. Repo anchor: [transaction middleware](../../server/middlewares/transaction.ts); the threading discipline; afterCommit hooks ([Event.ts L82–97](../../server/models/Event.ts#L82-L97)); the lone row lock ([comments.create](../../server/routes/api/comments/comments.ts#L28)).
Junior: "for multiple writes." Mid: middleware-per-mutation pattern; the unthreaded-query trap; side effects belong after commit. Senior: lock selectivity (only where concurrent writers corrupt merged state), long-transaction dangers (external calls, pool exhaustion), and isolation-level defaults you're actually relying on.

## Q7: "REST, RPC, or GraphQL — pick for a wiki product and defend it."
Round: API/data. Repo anchor: the RPC choice ([Pattern 1](../03-architecture-and-patterns/05-pattern-catalog.md)).
Junior: picks by fashion. Mid: operation-heavy domain → RPC verbs; REST's caching/status conventions lost — and why that's acceptable (authed app traffic, POST-everything with CSRF). Senior: GraphQL's real cost in this domain is per-field authorization over a policy engine — a specific, experienced objection; plus "the API is public product surface here, so verb clarity beats resource purity."

## Q8: "How do you version a public API without breaking users?"
Round: API/data. Repo anchor: header-based version switches (`x-api-version` changing response nesting/format — [documents.info L640–648](../../server/routes/api/documents/documents.ts#L640-L651), [presenter L38](../../server/presenters/document.ts#L38)).
Junior: /v2/ URLs. Mid: additive-first policy; version headers for shape changes; defaults never change silently ([Kata 4](../04-code-reading-gym/04-review-katas.md)). Senior: version *sunsetting* discipline, and the observation that every default is an implicit contract — with this repo's `data`/`text` document-body migration as the worked example.

## Q9: "Design soft delete. What breaks?"
Round: API/data. Repo anchor: paranoid models + restore + purge tasks ([Pattern 11](../03-architecture-and-patterns/05-pattern-catalog.md)).
Junior: `deletedAt` column, done. Mid: default-scope filtering, restore flows needing `paranoid: false`, unique-constraint collisions, cascade questions. Senior: retention/compliance (purge scheduling — [CleanupDeletedDocumentsTask](../../server/queues/tasks/CleanupDeletedDocumentsTask.ts)), and the client-side mirror (paranoid client models set `deletedAt` locally — [Store.remove L214–218](../../app/stores/base/Store.ts#L174-L221)).

## Q10: "How do you make an endpoint idempotent?"
Round: API/data. Repo anchor: client-supplied ids (comments.create accepts an `id` — [comments.ts](../../server/routes/api/comments/comments.ts#L28)); consumer-side guards ([RevisionsProcessor equality check](../../server/queues/processors/RevisionsProcessor.ts#L34-L44)); unique constraints as the backstop.
Junior: "check if it exists first" (race!). Mid: idempotency keys / natural keys + upsert-or-conflict semantics; retries as the reason. Senior: the layered version (client dedupe → key → constraint) and *which operations don't need it* (naturally idempotent updates) — spending effort proportional to stakes.

## Q11: "Full-text search: build it into Postgres or run Elasticsearch?"
Round: API/data. Repo anchor: [Flow 4](../01-codebase-cartography/05-key-flows.md); tsvector default with a provider seam + index events already emitted.
Junior: "Elastic is for search." Mid: Postgres FTS goes far (no infra, transactional consistency); external engines buy ranking/scale at the cost of sync pipelines. Senior: the migration path (backfill/shadow/cutover — [SP-4](../06-contribution-practice/03-senior-build-projects.md)) and permission filtering *inside the index query* as the security crux.

## Q12: "Design audit logging."
Round: API/data. Repo anchor: the events table doing triple duty (audit + bus trigger + client-visible activity — [Event.ts](../../server/models/Event.ts)).
Junior: "log to a file." Mid: structured rows (actor, ip, authType, target ids), written transactionally with the change. Senior: the coupling critique — audit (immutable, complete) vs bus (transient, at-least-once) vs UI feed (presentable) have different requirements, and this repo's `Event.schedule` (bus-only, *no* audit row — [L136–148](../../server/models/Event.ts#L136-L148)) shows the tension; retention and PII policy (`data` is client-visible!) close the answer.

---

Highest-yield for mid-level loops: Q2, Q4, Q5, Q6. If you can deliver those four at "Strong," you will out-answer most mid-level candidates on this round.
