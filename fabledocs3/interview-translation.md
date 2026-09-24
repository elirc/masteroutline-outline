# Interview Translation

Each ADR compressed into what you'd actually *say* in an interview — roughly
90 seconds spoken, first person, tradeoffs included (senior candidates volunteer
the costs; juniors wait to be asked). After each answer: the follow-up an
interviewer is most likely to hit you with, so you're not improvising.

Delivery notes that apply to all ten: name the constraint first, then the
decision, then one alternative you rejected *and why*, then the cost you
knowingly accepted. That four-beat structure is what "staff-level answer"
sounds like.

---

## ADR-001 — Modular monolith with service flags

**Spoken answer:**
"We shipped to two audiences at once — self-hosters who want one Docker
container, and our cloud which needs to scale websockets, workers, and the
collaborative editing server independently. So instead of microservices, we
built one artifact where every service — web, worker, websockets,
collaboration, cron — is a module, and a `--services` flag picks what a given
container runs. Self-hosters run everything in one process; our cloud runs
the same image as specialized pods. The big win is that all services share
the same models, authorization code, and editor schema by direct import —
there's no cross-service API to version, which matters enormously because the
collaboration server and the API must agree on the document schema exactly.
The cost we accepted: no fault isolation on single-container installs — a
memory leak in collaboration takes the API down with it. And there's one
sharp edge: collaboration state is in-memory, so without a Redis sync URL we
have to clamp to one process, and we do that with a quiet runtime check
rather than refusing to boot, which I'd do differently today."

**Likely follow-up:** "At what point would you actually split a service out
into its own deployable, and which one goes first?"

---

## ADR-002 — Three-runtime layering (`app`/`server`/`shared`)

**Spoken answer:**
"We split the codebase by runtime, not by feature: browser code, server code,
and a `shared` tree that runs in both. That sounds mundane, but for a
collaborative editor it's the load-bearing decision — the Prosemirror
document schema lives in `shared`, and both the browser editor and the server
that merges concurrent edits import the *same* schema. If those ever
diverged, that's not a bug, that's data corruption. We enforce the
dual-runtime property in CI by running the shared tests twice — once under
Node, once under jsdom — so a server-only import into shared fails visibly.
Optional features are a fourth tree: compile-time plugins registered against
typed hooks, so core code never imports an integration by name. We rejected
feature-sliced packages because the hard boundary in this system is 'which
runtime can execute this,' and we rejected runtime-loadable third-party
plugins because a plugin that can touch models can corrupt data, and
sandboxing Node wasn't a feature-sized problem. The tax is that one feature's
logic spans three directories, which every new hire has to internalize."

**Likely follow-up:** "How do you stop someone importing server code into
`shared` — is it convention or is it enforced?"

---

## ADR-003 — RPC-over-POST API

**Spoken answer:**
"Our public API is RPC, not REST — everything is `POST /api/documents.update`
style, noun-dot-verb, parameters in the body. That was deliberate: the
domain's operations are verbs like 'move this document and reposition its
siblings' or 'restore this revision,' and forcing those into REST resource
semantics just generates design meetings. Named operations also make
per-operation rate limits and audit logs trivial. Every route declares a Zod
schema colocated with the handler, and a validation middleware parses the
request before the handler ever sees it — handlers never touch raw bodies.
We looked hard at GraphQL because the frontend wants nested data, but our
authorization is per-object and conditional, and resolver graphs make it too
easy to leak a field through a path nobody authorized; plus 'POST this JSON
here' is a better contract for integration customers than a query language.
The honest costs: we over-fetch — our presenter layer keeps growing option
flags that are really field-selection reinvented one boolean at a time — and
all-POST means we gave up HTTP caching on reads."

**Likely follow-up:** "You gave up HTTP caching — so how do you handle read
performance on hot endpoints?"

---

## ADR-004 — Centralized policy layer, shipped to the client

**Spoken answer:**
"All authorization goes through one CanCan-style policy engine — per-model
policy files declaring abilities as TypeScript predicates, and handlers call
`authorize(user, action, object)`, which throws on failure. So authorization
is an assertion, not a query you can forget to check. The part I'd highlight:
we serialize the *same* ability evaluations into API responses, and the
frontend renders UI from them — whether you see a Delete button and whether
DELETE succeeds are decided by the same code. That kills the whole class of
frontend-backend permission drift. We rejected inline role checks — that's
how the same rule gets written four different ways — and we rejected Postgres
row-level security, which is real defense-in-depth but miserable to debug
through an ORM, and our self-hosters would blame their own database config
for every permission bug. The residual risk I'd name: policies evaluate on
loaded objects, so list endpoints' WHERE clauses have to stay manually
aligned with what policies would allow — two places encoding one rule."

**Likely follow-up:** "How do you keep list-query filters and policy
predicates from drifting apart over time?"

---

## ADR-005 — Sequelize with explicit transaction-per-mutation

**Spoken answer:**
"Every mutating endpoint runs inside a database transaction opened by
middleware, and the transaction handle is passed *explicitly* through every
command and query — we deliberately rejected AsyncLocalStorage-style implicit
propagation. The reasoning: when ambient context gets lost — a stray timer, a
library that drops async context — implicit transactions fail *silently into
autocommit*, which is the worst possible failure mode for a correctness
mechanism. Explicit passing is ugly but its failure is visible in code
review. This matters because our mutations are never single-row: updating a
document also writes an event row that websockets, webhooks, and search
indexing all consume — if those don't commit atomically, downstream consumers
starve silently. The costs I'll own: the transaction parameter is on
practically every function signature in the codebase, and forgetting to pass
it compiles fine and passes low-concurrency tests — enforcement is convention
and review, not the type system. And on the genuinely concurrency-critical
paths we drop below the ORM anyway — raw `SET LOCAL lock_timeout`, explicit
row locks — because that's where ORM abstractions stop being honest."

**Likely follow-up:** "Could you make forgetting the transaction a
compile-time or CI-time error instead of a review-time one?"

---

## ADR-006 — Events table + queues as the integration spine

**Spoken answer:**
"Every mutation writes an event row in the same transaction as the change,
and after commit we publish it onto Redis-backed queues where processors fan
out — websocket broadcasts, notifications, webhooks, search indexing,
revision snapshots are all *consumers of events*, not code called from route
handlers. So adding a side effect means adding a processor, not editing
existing endpoints, and API latency never couples to a Slack outage. The
event table doubles as a user-facing audit log, transactionally correct for
free. We rejected Kafka — better semantics, disqualifying ops burden for
self-hosters — and inline side effects for the coupling reasons. The two
weaknesses I'd volunteer: first, there's a commit-to-publish gap — the event
row commits, then we publish in-process, so a crash in between leaves an
event nothing ever consumed; it's an outbox in shape without the relay that
makes outboxes safe. Second, delivery is at-least-once from the queue on, so
every consumer has to be idempotent, and that discipline varies. If I were
rebuilding it, the relay is the first thing I'd add."

**Likely follow-up:** "Walk me through exactly how you'd add that relay
without double-delivering existing events."

---

## ADR-007 — Yjs CRDT with triple document representation

**Spoken answer:**
"Real-time editing runs on Yjs CRDTs behind a Hocuspocus server. We chose
CRDTs over operational transformation because OT needs a central server
sequencing every operation, while CRDTs tolerate offline edits and merge
safely on reconnect — a client that crashed can just resend its state. The
interesting part is storage: every document has *three* representations on
one row — the binary CRDT state, which is authoritative, plus Prosemirror
JSON and markdown as projections that refresh asynchronously, because search,
exports, and the public API predate collaboration and were built on text.
Migration was lazy, not big-bang: the first time a legacy document gets
opened for editing, we build its CRDT state from the markdown under a row
lock. We rejected the offline mass-conversion because a bug there corrupts
documents nobody opens for months — lazy conversion means failures surface
in front of an active user immediately. The permanent tax: every reader must
know which representation it's reading and how stale it can be — 'search
shows old text for a minute' is an accepted anomaly — and the lock
choreography has scars: we added a fifteen-second lock timeout after a large
document's save convoyed the connection pool."

**Likely follow-up:** "Why is the CRDT blob authoritative rather than the
JSON — couldn't you rebuild state from a snapshot?"

---

## ADR-008 — Redis as the horizontal-scale substrate

**Spoken answer:**
"Our rule was: self-hosters get exactly two pieces of infrastructure,
Postgres and Redis. So Redis wears every distributed-systems hat — socket.io
pub/sub so a broadcast from any worker reaches sockets on any web process,
Bull job queues, distributed locks for cache-stampede protection, rate-limit
counters keyed per-user so an office behind one NAT doesn't share a bucket,
and optionally syncing collaborative document state across nodes. We
considered doing it all in Postgres — LISTEN/NOTIFY, advisory locks — but
Postgres was already our most contended resource given the CRDT workload's
row locking, so loading it with fan-out traffic was the wrong direction. And
dedicated best-of-breed brokers per concern are individually better but
collectively an ops tax our self-hosted users won't pay. The tradeoff I'd
state plainly: Redis is a single point of failure wearing five hats — one
outage simultaneously degrades websockets, background work, rate limiting,
and cache, failure modes that separate infrastructure would isolate. We also
carry a degradation path for running without parts of it, and that
'works-without-it-but-worse' mode doubles the surface we have to test."

**Likely follow-up:** "Redis goes down for ten minutes in production — what
does the user actually experience, subsystem by subsystem?"

---

## ADR-009 — Error philosophy: stable ids and the reportable bit

**Spoken answer:**
"Errors have one vocabulary and one exit. The vocabulary is a set of factory
functions — every error carries an HTTP status, a stable machine-readable id
like `authorization_error` that clients branch on so message text can change
and be translated freely, and an `isReportable` boolean. That bit is the
design: it's set where the error is *constructed*, because the author knows
whether a 403 is boring traffic or a genuine bug, and a generic catch
handler guessing from status codes knows the least. The exit is Koa's single
onerror funnel — handlers throw early and don't catch — and only reportable
errors go to Sentry, which is why our error tracker contains signal instead
of noise. We had the Result-types debate and rejected it: Koa's middleware
model *is* exception propagation, and Result-typed internals inside an
exception-shaped framework means lossy conversion at every boundary. The
cost is that TypeScript can't tell you what a function throws, so error
contracts live in docs and tests. And the reportable bit has a sharp inverse
failure: mislabel one frequent error as reportable and you flood Sentry into
rate-limiting right when a real regression needs it — one boolean, big blast
radius."

**Likely follow-up:** "How do you catch a *mislabeled* error — one marked
non-reportable that's actually a real bug?"

---

## ADR-010 — Integration-first testing on real infrastructure

**Spoken answer:**
"Our server tests boot the real Koa app and make real HTTP calls against
real Postgres — middleware, validation, authorization, presenters all in the
loop — with outbound third-party HTTP as the only mocked seam. We rejected
mocking the ORM because those tests assert your *beliefs* about the ORM, and
beliefs drift: a Sequelize major upgrade once changed soft-delete scope
behavior, and only real-database tests caught deleted documents reappearing
in a list endpoint. We rejected SQLite because we lean on Postgres-specific
behavior — row locks, tsvector search — so SQLite-green means nothing. Test
data comes from factories, not fixtures, and we have a standing convention
that every authorization test includes an actor from a *second* team,
because cross-tenant leakage is the bug class a multi-tenant-by-column
system can least afford. Costs, honestly: CI takes minutes and needs service
containers; isolation is by cleanup convention rather than rollback-per-test
— because the code under test manages its own transactions — so occasional
state leaks between tests are miserable to bisect. And our biggest gap is
the composition: multi-client concurrent editing has no automated end-to-end
harness; we test the pieces and lean on CRDT guarantees plus staged
rollouts."

**Likely follow-up:** "If you had one week to close a testing gap, do you
build the multi-client collaboration harness or fix test isolation — and
why?"

---

## One last coaching note

Interviewers probe the *rejected* alternatives harder than the decision. For
every ADR above, be ready to argue the other side convincingly for thirty
seconds — "here's the world where GraphQL was the right call" — and then say
why that world isn't this one. Knowing why the road not taken was real is
what separates "I read the codebase" from "I could have made the decision."
