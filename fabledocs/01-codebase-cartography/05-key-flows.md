# Key Flows

Six end-to-end traces. These are the backbone of the whole curriculum — later modules (drills, tickets, interview cards) rotate through them rather than re-explaining. Line numbers verified at commit `8e032f8cd`.

---

## Flow 1: Renaming a document (client → API → DB → event bus → every other client)

Why this flow matters: it exercises every layer once — the canonical "walk me through a request" answer.

Open these files first:
- [app/models/base/Model.ts](../../app/models/base/Model.ts) — client `save()` produces the RPC call
- [server/routes/api/documents/documents.ts#L1251-L1294](../../server/routes/api/documents/documents.ts#L1251-L1294) — the endpoint
- [server/commands/documentUpdater.ts](../../server/commands/documentUpdater.ts) — the mutation
- [server/models/Event.ts#L82-L97](../../server/models/Event.ts#L82-L97) — the sync→async hinge
- [app/components/WebsocketProvider.tsx#L253-L262](../../app/components/WebsocketProvider.tsx#L253-L262) — the loop closing

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | UI | app/scenes/Document | User edits title; model `save()` called | `{id, title}` partial | Debounce/stale-closure bugs |
| 2 | Client transport | [app/utils/ApiClient.ts](../../app/utils/ApiClient.ts#L106) | `POST /api/documents.update`, JSON body, CSRF cookie echoed, in-flight dedupe | JSON body | Duplicate submits (deduped by key) |
| 3 | API middleware | [documents.ts L1251–1254](../../server/routes/api/documents/documents.ts#L1251-L1254) | `auth()` → `validate(DocumentsUpdateSchema)` → `transaction()` | `ctx.state.auth`, `ctx.input.body`, `ctx.state.transaction` | Middleware order is load-bearing |
| 4 | Data access | [Document.findByPk override](../../server/models/Document.ts#L758-L833) | Load doc **with membership scope** for this user, inside the txn | `Document` with `memberships`, `collection` preloaded | Forgetting `userId` ⇒ policy invariant blows up later |
| 5 | Authorization | [documents.ts L1270](../../server/routes/api/documents/documents.ts#L1264-L1278) + [document policy](../../server/policies/document.ts#L86-L102) | `authorize(user, "update", document)` throws 403 if not allowed | — | Authz *after* validation, *before* mutation |
| 6 | Domain | [documentUpdater.ts L68–125](../../server/commands/documentUpdater.ts#L68-L125) | Mutate fields; if `changed()`, `saveWithCtx` persists and records a `documents.update` Event in the same txn | `Document` row + `events` row | Event and row committed atomically |
| 7 | Commit hinge | [Event.ts L82–97](../../server/models/Event.ts#L82-L97) | `@AfterSave` registers `afterCommit` → enqueue to Bull `globalEvents` | Event JSON on Redis | If enqueue were pre-commit, workers could see uncommitted data |
| 8 | Worker | [worker.ts L53–88](../../server/services/worker.ts#L53-L88) | Fan out: each processor whose `applicableEvents` matches gets its own job; websockets job goes to a dedicated queue | `{event, name}` per processor | Per-processor isolation: one failing processor retries alone |
| 9 | Realtime | [WebsocketsProcessor](../../server/queues/processors/WebsocketsProcessor.ts) via [websockets service](../../server/services/websockets.ts) | Emit `documents.update` to the document/collection rooms | Presented partial document | Clients on old code receive new payloads — wire compat matters |
| 10 | Other clients | [WebsocketProvider L253–262](../../app/components/WebsocketProvider.tsx#L253-L262) | `documents.add(event)` merges into the observable cache; MobX re-renders observers | Partial model | Merge must tolerate partials |

Validation and authorization: schema in [documents/schema.ts](../../server/routes/api/documents/schema.ts) (`DocumentsUpdateSchema`); policy chain `update` → `read` in [document.ts L21–102](../../server/policies/document.ts#L21-L102).

Persistence and side effects: one transaction covers steps 4–7's writes ([transaction.ts](../../server/middlewares/transaction.ts#L13-L20)). Post-commit: revisions (debounced), search index, webhooks, notifications — all via the same event.

Tests that cover it: [server/routes/api/documents/documents.test.ts](../../server/routes/api/documents/documents.test.ts), [server/commands/documentUpdater.test.ts](../../server/commands/documentUpdater.test.ts).

What juniors usually miss: the event is not "sent" by the endpoint — it's a model hook + afterCommit; and the responding client *also* gets the websocket event (client code must handle self-echoes; see the "was it us?" check around [WebsocketProvider L245+](../../app/components/WebsocketProvider.tsx#L245-L262)).

What seniors notice: `documentUpdater` has three exit paths (publish / changed / done-only Event.schedule) — the `done` path at [L119–125](../../server/commands/documentUpdater.ts#L119-L125) emits an event *without* writing, which feeds the revision debounce. Also: the final re-fetch at [L127–131](../../server/commands/documentUpdater.ts#L127-L131) inside the txn guarantees the response reflects committed state with proper scopes.

Interview angle: "Walk me through a write request in your stack" and "how do you keep cache/live views consistent after writes?" — this flow is a complete, concrete answer.

Drill: set a breakpoint (or add temporary logs) at steps 3, 6, 8, 10 and rename a document. Write down the actual payload at each step, then delete the logs.

Self-grade — Basic: name the layers in order. Solid: explain the transaction/afterCommit boundary and why it exists. Strong: explain what happens if Redis is down at step 7 (request succeeds; event enqueue fails post-commit — silent loss; see risk register) and how you'd detect it.

---

## Flow 2: Anonymous user opens a public share link (auth/security boundary)

Why this flow matters: it's the repo's sharpest **authorization boundary** — unauthenticated access to selected private data, with several subtle protections.

Open these files first:
- [server/commands/shareLoader.ts#L23-L70](../../server/commands/shareLoader.ts#L23-L70) — what makes a share valid (`published: true`, `revokedAt: null`)
- [documents.ts L574–652 (`documents.info`)](../../server/routes/api/documents/documents.ts#L574-L652)
- [server/presenters/document.ts#L97-L129](../../server/presenters/document.ts#L97-L129) — field withholding for public viewers

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | Browser | `/s/:shareId` route | SPA loads share scene; ApiClient gets `shareId` appended to requests ([ApiClient.setShareId](../../app/utils/ApiClient.ts#L78-L80)) | — | — |
| 2 | API | [documents.info L575](../../server/routes/api/documents/documents.ts#L574-L583) | `auth({ optional: true })` — user may be null | — | Optional auth: every later line must handle `user == null` |
| 3 | Share resolution | [shareLoader L23–70](../../server/commands/shareLoader.ts#L23-L70) | Find share by id/urlId where `published` and not revoked; load doc + collection structure | `Share` + `Document` | urlId lookups additionally require `teamId` (multi-tenant slug collisions) |
| 4 | Publicness decision | [documents.info L602–610](../../server/routes/api/documents/documents.ts#L602-L610) | If a user is signed in, reload doc with their memberships; then `isPublic = cannot(user, "read", document)` | boolean | Elegant: "public rendering" = "you couldn't see it privately" |
| 5 | Scoped presentation | [presenter L40–99](../../server/presenters/document.ts#L38-L99) | Public docs: signed attachment URLs, comment marks stripped (`removeMarks: ["comment"]`), internal links rewritten under `/s/:shareId`, `updatedAt` withheld unless share opts in | Reduced JSON | Leaking fields here = data disclosure |
| 6 | Backlinks fencing | [documents.info L612–620](../../server/routes/api/documents/documents.ts#L612-L620) | Backlinks restricted to ids inside the shared tree | `backlinkIds` | Prevents enumerating private docs via backlinks |
| 7 | Policies omitted | [documents.info L649](../../server/routes/api/documents/documents.ts#L640-L650) | `policies: undefined` for public viewers | — | — |

Validation and authorization: schema `DocumentsInfoSchema` (requires id or shareId); the whole flow is authorization-by-construction — anonymous users never hit `authorize`, they only receive what the share tree exposes.

Persistence and side effects: read-only; view counting happens on other endpoints.

Tests that cover it: [documents.test.ts](../../server/routes/api/documents/documents.test.ts) (share cases), [shareLoader.test.ts](../../server/commands/shareLoader.test.ts).

What juniors usually miss: search on shares is separately fenced — `documents.search` refuses child-doc search when `includeChildDocuments` is off ([documents.ts L1088–1090](../../server/routes/api/documents/documents.ts#L1080-L1095)).

What seniors notice: the comment-mark stripping in the presenter is defense against leaking *comment metadata* through document content — presenters as a security layer, not just formatting. Also the signed-URL TTL (`Hour.seconds`) bounds how long a leaked payload stays useful.

Interview angle: "How would you add public sharing to a private multi-tenant app?" — this flow gives you the checklist: explicit share model with revocation, publicness computed not assumed, response minimization, link rewriting, sub-tree fencing.

Drill: list three distinct ways a private fact could leak through a naive share implementation, then find the specific line here that blocks each.

Self-grade — Basic: explain `auth({optional})`. Solid: explain the `isPublic = cannot(...)` trick and field withholding. Strong: propose a new leak vector (e.g. embeds/unfurls inside shared docs) and how you'd test for it.

---

## Flow 3: SSO sign-in and account provisioning

Why this flow matters: authentication (who are you) as distinct from authorization (what may you do), plus idempotent user/team provisioning — interview staples.

Open these files first:
- [server/routes/auth/index.ts](../../server/routes/auth/index.ts) — mounts one passport router per provider from `AuthenticationHelper.providers` (plugins register themselves)
- [server/commands/accountProvisioner.ts#L91-L215](../../server/commands/accountProvisioner.ts#L91) — find-or-create of team, user, authentication
- [server/middlewares/passport.ts](../../server/middlewares/passport.ts) — OAuth state cookie handling

Trace:

| Step | Owner | File | What happens | Risk |
| --- | --- | --- | --- | --- |
| 1 | Browser | login scene | Redirect to `/auth/<provider>` | Open-redirect care in `state` |
| 2 | Provider plugin | e.g. [plugins/google/server](../../plugins/google/server) | Passport strategy exchanges code for profile | Provider trust; email verification flag |
| 3 | Provisioning | [accountProvisioner](../../server/commands/accountProvisioner.ts#L91) → [teamProvisioner](../../server/commands/teamProvisioner.ts), [userProvisioner](../../server/commands/userProvisioner.ts) | Match team by auth-provider id (or domain rules); find-or-create user; record `users.signin`-type events | **Idempotency**: repeated logins must not duplicate; email collisions across providers |
| 4 | Session | JWT set as `accessToken` cookie | Subsequent requests take the JWT branch of [authentication.ts L242–247](../../server/middlewares/authentication.ts#L242-L247) | Cookie flags, expiry |
| 5 | Suspension/role gates | [authentication.ts L249–260](../../server/middlewares/authentication.ts#L249-L260) | Suspended users rejected with admin contact; role floor enforced per-route | — |

Validation and authorization: this flow *creates* the identity that all policies later consume. Note `ValidateSSOAccessTask` in [server/queues/tasks](../../server/queues/tasks/ValidateSSOAccessTask.ts) — periodic re-validation that SSO access still holds (offboarding safety), a nice "senior" detail.

Persistence and side effects: team/user/authentication rows; welcome/onboarding docs seeded from [server/onboarding](../../server/onboarding) markdown (see accountProvisioner's imports of `DocumentHelper`/onboarding files).

Tests that cover it: [accountProvisioner.test.ts](../../server/commands/accountProvisioner.test.ts), [teamProvisioner.test.ts](../../server/commands/teamProvisioner.test.ts), [userProvisioner.test.ts](../../server/commands/userProvisioner.test.ts), route tests in [server/routes/auth/index.test.ts](../../server/routes/auth/index.test.ts).

What juniors usually miss: three token types exist (cookie JWT for the app, API keys, OAuth access tokens for third-party apps) with different allowed transports — [authentication.ts L156–241](../../server/middlewares/authentication.ts#L156-L241) — and those restrictions are security decisions, not pedantry.

What seniors notice: provisioning runs in its own transaction boundaries and emits audit events via `Event.createFromContext` ([accountProvisioner L210](../../server/commands/accountProvisioner.ts#L210)); the plugin system means auth providers are additive without touching core — an **open/closed** boundary.

Interview angle: "Design login with Google for a multi-tenant SaaS" — answer with: provider abstraction, provisioning idempotency, domain-based team matching, suspended/role gates, session transport choices, and periodic SSO revalidation.

Drill: read `userProvisioner` and write down every distinct outcome (new user, existing user new provider, existing invite claimed, banned domain, …). Compare with its test file.

Self-grade — Basic: name authn vs authz. Solid: describe find-or-create idempotency and why email alone is a dangerous match key. Strong: explain how you'd migrate a team from Slack SSO to Google SSO without lockout.

---

## Flow 4: Full-text search

Why this flow matters: pluggable provider behind an interface, rate limiting, tenant scoping, and telemetry (SearchQuery rows) in one endpoint.

Open these files first:
- [documents.ts L1034–1120 (`documents.search`)](../../server/routes/api/documents/documents.ts#L1034-L1120)
- [server/utils/SearchProviderManager.ts](../../server/utils/SearchProviderManager.ts) — provider selection
- [plugins/search-postgres](../../plugins/search-postgres) — default implementation (tsvector; `pg-tsquery` sanitizes user syntax)
- [SearchIndexProcessor](../../server/queues/processors/SearchIndexProcessor.ts#L21-L46) — index maintenance events (no-op for Postgres, real for external providers)

Trace:

| Step | Owner | File | What happens | Risk |
| --- | --- | --- | --- | --- |
| 1 | API | [documents.search L1035–1039](../../server/routes/api/documents/documents.ts#L1034-L1040) | `auth({optional}) → pagination() → rateLimiter(100/min) → validate` | Search is expensive ⇒ rate limited |
| 2 | Two modes | L1063–1120 | Share mode (public, fenced to share tree) vs authenticated mode (team-scoped) | Same duality as Flow 2 |
| 3 | Provider | `SearchProviderManager.getProvider().searchForTeam(team, {...})` | Query with collection filter, status filter, snippet params, popularity boost | Provider contract = interface **contract** |
| 4 | Telemetry | `SearchQuery` row saved with duration (see [SearchQuery model](../../server/models/SearchQuery.ts)) | Powers "no results" analytics | PII in queries — retention matters |
| 5 | Client | [app/scenes/Search](../../app/scenes/Search) + [SearchesStore](../../app/stores/SearchesStore.ts) | Debounced input, paginated results | Stale result races |

Validation and authorization: collection filter re-authorizes (`Collection.findByPk(..., {userId})` then policy) — user can't search collections they can't read.

Persistence and side effects: SearchQuery insert; index maintenance is asynchronous via events (external providers), tsvector triggers (Postgres).

Tests that cover it: search cases in [documents.test.ts](../../server/routes/api/documents/documents.test.ts); [SearchIndexProcessor.test.ts](../../server/queues/processors/SearchIndexProcessor.test.ts).

What juniors usually miss: results must be permission-filtered *in the query*, not post-filtered in JS — post-filtering breaks pagination and leaks counts.

What seniors notice: the provider interface lets Postgres FTS be swapped for Elasticsearch without touching the route — and the processor exists *now*, subscribed to every content event, so the index stays consistent when a provider that needs it is enabled. Building the event plumbing before you need it is a deliberate tradeoff.

Interview angle: "How would you add search to your app?" and the follow-up "Postgres FTS or Elasticsearch?" — answer with this repo's actual shape: start with Postgres tsvector (zero extra infra, good enough), keep a provider seam, emit index events from day one.

Drill: find where the Postgres provider applies the user's collection permissions (grep `collectionIds` in [plugins/search-postgres](../../plugins/search-postgres)); write the one-sentence invariant it maintains.

Self-grade — Basic: list the middleware chain. Solid: explain query-time permission filtering vs post-filter. Strong: sketch the migration plan to an external index (backfill, dual-write, cutover, deletes).

---

## Flow 5: Revision history (background/async flow)

Why this flow matters: it's the repo's cleanest example of **eventual consistency by design** — revisions are deliberately not written during the request.

Open these files first:
- [RevisionsProcessor.ts](../../server/queues/processors/RevisionsProcessor.ts) — trigger conditions
- [server/commands/revisionCreator.ts](../../server/commands/revisionCreator.ts)
- [DebounceProcessor.ts](../../server/queues/processors/DebounceProcessor.ts) — produces `documents.update.debounced` (*investigate: read this to confirm exact delay semantics*)

Trace:

| Step | Owner | File | What happens | Risk |
| --- | --- | --- | --- | --- |
| 1 | Edits | Flows 1/collab | Many `documents.update` events, most with `done` unset | Naive design would write a revision per keystroke |
| 2 | Debounce | DebounceProcessor | Collapses bursts into `documents.update.debounced` | Delay = revision granularity |
| 3 | Guard | [RevisionsProcessor L20–23](../../server/queues/processors/RevisionsProcessor.ts#L17-L24) | Plain `documents.update` events only proceed when `data.done` | |
| 4 | Collaborators | [L26–29](../../server/queues/processors/RevisionsProcessor.ts#L25-L29) | Redis set of collaborator ids since last revision, read-and-cleared | Non-transactional Redis handoff — crash between `smembers` and revision write loses attribution (possible risk, small) |
| 5 | Idempotency-ish guard | [L36–44](../../server/queues/processors/RevisionsProcessor.ts#L34-L44) | Skip if content equals previous revision | Dedup by comparison, not by key |
| 6 | Write | revisionCreator | Insert Revision + `revisions.create` event | Feeds notifications (Flow 6 pattern) |

Persistence and side effects: Revision row; downstream `revisions.create` triggers [RevisionCreatedNotificationsTask](../../server/queues/tasks/RevisionCreatedNotificationsTask.ts).

Tests that cover it: [RevisionsProcessor.test.ts](../../server/queues/processors/RevisionsProcessor.test.ts), [revisionCreator.test.ts](../../server/commands/revisionCreator.test.ts).

What juniors usually miss: Bull retries (5 attempts, exponential backoff — [queues/index.ts](../../server/queues/index.ts)) mean `perform` can run more than once → every processor must be safe to re-run (**idempotency**). Here the equality check at step 5 doubles as the idempotency guard.

What seniors notice: "read-and-delete a Redis set, then write Postgres" is a tiny distributed transaction with no atomicity. Acceptable because the failure cost is mis-attributed collaborators, not data loss — matching consistency guarantees to stakes is exactly the judgment interviews probe.

Interview angle: "How would you implement document version history?" — answer: not synchronously; debounce edit sessions, snapshot on session end, dedupe identical content, keep attribution best-effort.

Drill: write the failure matrix — for each step, "what happens if the process dies here?" Three columns: lost?, duplicated?, acceptable-why?

Self-grade — Basic: why async. Solid: retries ⇒ idempotency requirement, and where the guard is. Strong: the Redis/Postgres atomicity gap and why it's an acceptable tradeoff.

---

## Flow 6: Creating a comment → notifications → email

Why this flow matters: a mutation with a **row lock**, content transformation, and a three-stage async pipeline (event → processor → tasks → email queue).

Open these files first:
- [comments.ts, `comments.create`](../../server/routes/api/comments/comments.ts#L28) — note the middleware adds `rateLimiter(25/min)` and a `commentingEnabled()` feature gate
- [NotificationsProcessor.ts L28–40](../../server/queues/processors/NotificationsProcessor.ts#L28-L41)
- [CommentCreatedNotificationsTask](../../server/queues/tasks/CommentCreatedNotificationsTask.ts) and [server/emails/templates/CommentCreatedEmail.tsx](../../server/emails/templates/CommentCreatedEmail.tsx)

Trace:

| Step | Owner | What happens | Risk |
| --- | --- | --- | --- |
| 1 | API | auth → feature gate → validate → transaction | Feature gates as middleware — reusable pattern |
| 2 | Lock | `SELECT … FOR UPDATE` on the document row when anchoring text ([comments.ts, inside comments.create](../../server/routes/api/comments/comments.ts#L28)) | Prevents two concurrent inline comments clobbering the doc's Yjs `state`. Classic **write-skew** prevention |
| 3 | Authz | `authorize(user, "comment", document)` → [policy](../../server/policies/document.ts#L53-L64) checks team preference + guest rules + collection flag | Multi-source permission — easy to miss one source |
| 4 | Content | Markdown text parsed to ProseMirror JSON; pasted images re-uploaded as attachments (`TextHelper.replaceImagesWithAttachments`) | User content = untrusted input |
| 5 | Anchor | `ProsemirrorHelper.applyCommentMarkByText` writes a comment mark into the doc state | Cross-model write: a comment mutates the *document* |
| 6 | Event | `comments.create` on the bus | |
| 7 | Notify | NotificationsProcessor → CommentCreatedNotificationsTask → Notification rows for subscribers/mentions (minus muted users, minus the actor) | Fan-out cost grows with subscribers |
| 8 | Email | Notification → email template rendered server-side (React via oy-vey) → sent by the email queue ([EmailsProcessor](../../server/queues/processors/EmailsProcessor.ts) / [EmailTask](../../server/queues/tasks/EmailTask.ts)) | Rendering user content into email safely |
| 9 | Realtime | Websocket event updates open clients' [CommentsStore](../../app/stores/CommentsStore.ts) | |

Tests that cover it: [comments.test.ts](../../server/routes/api/comments/comments.test.ts), notification task tests like [GroupMentionedInCommentNotificationsTask.test.ts](../../server/queues/tasks/GroupMentionedInCommentNotificationsTask.test.ts).

What juniors usually miss: the actor must not be notified about their own comment, and muted/unsubscribed users are filtered in the task, not the processor — the *task* owns recipient computation.

What seniors notice: step 2's explicit lock is rare in this codebase — locks appear only where two writers can corrupt a merged artifact (the Yjs state). Everywhere else optimistic behavior + last-write-wins is accepted. That selectivity is the lesson.

Interview angle: "Design comments with @-mentions and email notifications" — trace this pipeline: synchronous minimal write, async recipient resolution, template rendering, and dedupe/mute rules.

Drill: from `NotificationsProcessor.applicableEvents`, draw the full tree of tasks it can schedule for `comments.create` vs `revisions.create`. Then find where "don't email the actor" is enforced.

Self-grade — Basic: name the pipeline stages. Solid: explain the row lock's purpose. Strong: estimate the fan-out cost for a 1,000-subscriber doc and propose where you'd batch.
