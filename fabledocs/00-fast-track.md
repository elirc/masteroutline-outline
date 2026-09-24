# Fast Track: One Weekend with Outline

Goal: by Sunday night you can run Outline locally, explain two end-to-end flows out loud, and have made (and tested) one safe change. This page skips almost everything — what it skips is listed at the bottom.

## 1. Run it (Saturday morning)

All commands are **inferred** from [package.json](../package.json#L6-L39), the [Makefile](../Makefile), and [docker-compose.yml](../docker-compose.yml) — they were read, not executed, while writing this. Prerequisites: Node 20/22 (see `engines` in [package.json](../package.json#L44-L46)), Yarn 4 (`packageManager` field), Docker.

```bash
# Start Postgres + Redis, install deps, generate local SSL, run backend+frontend in watch mode
make up

# …or the manual equivalent:
docker compose up -d redis postgres
yarn install --immutable
yarn install-local-ssl
yarn dev:watch
```

You'll need a `.env` file; copy from `.env.sample` if present, and note [server/env.ts](../server/env.ts) is the single validated source of server configuration (it uses class-validator decorators — misconfigured env fails fast at boot). The dev server prints the URL it's listening on — see the "listening" handler in [server/index.ts](../server/index.ts#L208-L218).

Sanity checks once running:

- `GET /_health` returns `OK` — it pings Postgres and Redis: [server/index.ts](../server/index.ts#L159-L177).
- The web app loads and you can sign in (in dev, an email/magic-link style flow or a configured OAuth provider; see [server/routes/auth/index.ts](../server/routes/auth/index.ts) which mounts one router per registered provider).

## 2. Learn the shape (Saturday midday)

One paragraph to hold in your head: **the client is a MobX cache of server models; the server is an RPC API where every route is a middleware chain ending in a handler; every mutation writes rows and emits an Event; background processors turn Events into revisions, notifications, websocket pushes, and search updates.** Everything else is detail.

The first 10 files to open, in order:

1. [docs/ARCHITECTURE.md](../docs/ARCHITECTURE.md) — the maintainers' own map. 2 minutes.
2. [server/index.ts](../server/index.ts) — how one binary becomes 6 services via the `SERVICES` env var (L182–190).
3. [server/routes/api/documents/documents.ts](../server/routes/api/documents/documents.ts#L1251-L1294) — read just `documents.update`: `auth() → validate() → transaction() → handler`. This chain is the whole API design in one screen.
4. [server/middlewares/validate.ts](../server/middlewares/validate.ts) — 30 lines; Zod schema → `ctx.input`.
5. [server/middlewares/transaction.ts](../server/middlewares/transaction.ts) — 20 lines; one transaction wraps the handler.
6. [server/policies/cancan.ts](../server/policies/cancan.ts#L182-L191) — `authorize()` throws; then skim [server/policies/document.ts](../server/policies/document.ts#L86-L102) for what "can update a document" actually means.
7. [server/models/Event.ts](../server/models/Event.ts#L82-L97) — the `@AfterSave` hook that enqueues every audit event *after the transaction commits*. This is the hinge between sync and async worlds.
8. [server/services/worker.ts](../server/services/worker.ts#L53-L88) — how one event fans out to many processors.
9. [app/stores/base/Store.ts](../app/stores/base/Store.ts#L56-L96) — the client-side generic store: an observable `Map` of models plus CRUD methods.
10. [app/components/WebsocketProvider.tsx](../app/components/WebsocketProvider.tsx#L253-L262) — where server events land back in the client stores, closing the loop.

## 3. Trace two flows out loud (Saturday afternoon)

Do these as **teach-backs**: explain each flow aloud, as if an interviewer asked "walk me through what happens when…". Notes allowed; reading the code aloud is not.

**Flow A — a user renames a document.** Client calls `documents.update` → [documents.ts L1251](../server/routes/api/documents/documents.ts#L1251-L1294) authenticates, validates, opens a transaction, loads the doc *scoped to the user's memberships* (`Document.findByPk(id, { userId })` — see the override in [server/models/Document.ts](../server/models/Document.ts#L758-L833)), authorizes `update`, calls [documentUpdater](../server/commands/documentUpdater.ts#L45-L132) which mutates fields and saves with an event; after commit the event hits the queues ([Event.ts L82–97](../server/models/Event.ts#L82-L97)); the worker fans it out ([worker.ts L53–88](../server/services/worker.ts#L53-L88)); the WebsocketsProcessor pushes `documents.update` to team rooms; every open client's [WebsocketProvider](../app/components/WebsocketProvider.tsx#L253-L262) merges it into `DocumentsStore`. Full version: [01-codebase-cartography/05-key-flows.md](01-codebase-cartography/05-key-flows.md).

**Flow B — authorization on read.** `documents.info` at [documents.ts L574](../server/routes/api/documents/documents.ts#L574-L652) works for three kinds of caller: anonymous with a public `shareId`, authenticated user, or both. Notice `auth({ optional: true })`, and that "is this public?" is literally computed as `isPublic = cannot(user, "read", document)` (L610). Read the `read` policy in [document.ts](../server/policies/document.ts#L21-L34) and follow one branch: membership → draft-author → collection access.

## 4. One safe change + one test (Sunday)

Pick the smallest real thing: e.g. in [server/presenters/document.ts](../server/presenters/document.ts) add a temporary field, watch it appear in the API response, then revert. Or take Ticket 1 from [06-contribution-practice/01-good-first-tickets.md](06-contribution-practice/01-good-first-tickets.md).

Then run one focused test file (**inferred** commands; test DB setup is in the Makefile `test` target):

```bash
make test            # one-time: creates + migrates the test database, runs everything (slow)
yarn test server/commands/documentUpdater.test.ts    # the fast loop you'll actually use
```

Open [server/commands/documentUpdater.test.ts](../server/commands/documentUpdater.test.ts) and read how it uses factories from [server/test/factories.ts](../server/test/factories.ts) — `buildDocument`, `buildUser`, etc. — against a real test database. This repo's server tests are integration-style: real Postgres, mocked externals.

## 5. Final teach-back

Answer these aloud, 90 seconds each (this is the interview format):

1. "Walk me through what happens end-to-end when a user edits a document title." (Flow A)
2. "How does Outline decide whether you're allowed to see a document?" (Flow B)
3. "Why does the event enqueue happen in `afterCommit` and what would break if it didn't?" (Answer: the worker could process an event for a row whose transaction later rolls back — a consistency bug. See [Event.ts L87–95](../server/models/Event.ts#L82-L97).)

Self-grade: **Basic** — you name the files. **Solid** — you trace request → middleware chain → command → event → processor without notes. **Strong** — you also explain *why* each boundary exists (validation vs authorization vs transaction) and one failure mode of each.

## What the fast path skips

The editor (ProseMirror/Yjs — a world of its own under [shared/editor](../shared/editor)), collaborative editing internals ([server/collaboration](../server/collaboration)), imports/exports, the plugin system ([plugins/](../plugins)), OAuth provider internals, emails, search, performance, and all of the drills. That's what the rest of [the curriculum](README.md) is for.
