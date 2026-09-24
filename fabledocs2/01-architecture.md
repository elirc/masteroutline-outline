# 01 — Architecture: One Image, Six Services, and an Events Table

## What it is and why it exists

The whole product — API, frontend assets, websockets, the collaborative
editing server, background workers, and cron — compiles into one deployable
unit. At boot, `server/index.ts` reads a `--services=` flag (or the `SERVICES`
env var) and starts whichever subset you asked for. The service registry is
`server/services/index.ts`, and each entry (`web.ts`, `websockets.ts`,
`collaboration.ts`, `worker.ts`, `cron.ts`, `admin.ts`) is just a function
that attaches itself to the shared Koa app and HTTP server.

Why a monolith with flags instead of separate deployables? Because we ship to
two audiences with opposite needs. Self-hosters want `docker run` and done —
one container, one Postgres, one Redis. Our cloud wants horizontal scale —
dozens of `web` pods, a couple of `worker` pods, isolated `collaboration`
pods. The services flag gives both from one build artifact. `throng` in
`server/index.ts` handles per-container process fan-out on top of that.

The second load-bearing idea is the **event spine**. Nearly every mutation
writes a row to the `events` table (`server/models/Event.ts`) inside the same
transaction as the mutation, and after commit the event is pushed onto a Bull
queue (`server/queues/`). Processors (`server/queues/processors/`) subscribe
by event name and fan out into tasks (`server/queues/tasks/`). Websocket
broadcasts (`WebsocketsProcessor.ts`), notifications
(`NotificationsProcessor.ts`), search indexing (`SearchIndexProcessor.ts`),
revision snapshots (`RevisionsProcessor.ts`), and webhooks are all *consumers
of events*, not code called inline from route handlers. This is why the API
stays fast and why adding a new side effect rarely touches existing code.

The third idea is **plugins** (`plugins/`, loaded by
`server/utils/PluginManager.ts`). Anything optional — Slack, OIDC, S3 vs local
storage, the Postgres search provider — registers itself against a typed
`Hook` enum (API routers, auth providers, processors, tasks, email templates,
search providers). Core code iterates hooks; it never imports a plugin by name.

## The hard decisions

### 1. Monolith-with-flags vs. real microservices

We seriously considered splitting collaboration into its own repo and service
around the time we adopted Yjs. Real-time editing has a completely different
performance profile from the CRUD API — long-lived connections, memory-heavy
Y.Doc instances, CPU spikes on large documents. The argument for splitting was
isolation; the argument against was that the collaboration server needs the
*exact same* Prosemirror schema, models, and policy checks as the API, and
keeping two repos' schemas in lockstep is a version-skew factory.

We stayed monolithic and I still think that was right, but the tradeoff that
bothers me: a memory leak in the collaboration extension stack can take down
API traffic on a self-hosted single-container install, and there's nothing we
can do about it short of telling people to run `--services` splits — which
most self-hosters never read the docs to discover.

There's also a genuinely ugly constraint sitting in the first fifty lines of
`server/index.ts`: if you run the `collaboration` service without
`REDIS_COLLABORATION_URL`, we force `webProcessCount = 1`, because Yjs doc
state lives in process memory and two processes editing the same doc without
a Redis-backed sync would silently fork the document. A comment and a runtime
clamp is a weak defense for a data-integrity invariant. We'd design that
differently today — probably refuse to boot instead of quietly degrading.

### 2. The events table as the integration bus (vs. an actual message broker)

The alternatives were: call side effects inline (simple, but every new feature
slows down every API call and a Slack outage breaks document saves), or adopt
Kafka/NATS (durable, replayable, and a non-starter for self-hosters who
already grumble about needing Redis). We chose Postgres events + Bull/Redis
queues. Events give us an audit log for free (`server/routes/api/events/`
exposes it), and Bull gives retries.

What still bothers me: this is **at-least-once delivery with no outbox
discipline in the other direction** — the event row commits transactionally,
but queue publish happens after commit, so a crash in the gap means an event
row exists that no processor ever saw. It's rare and we've mostly ignored it;
"mostly" is doing work in that sentence. The `events` table also grows without
bound at cloud scale, which is why `CleanupOldEventsTask.ts` exists — it
wasn't in the original design, it was a remediation.

### 3. Plugins as first-class architecture (vs. a fork-friendly core)

Early on, integrations were `if (env.SLACK_KEY)` branches scattered through
core. That doesn't scale past three integrations, and it made the
enterprise/community split (see the BSL license and `plugins/enterprise`)
impossible to maintain cleanly. The rejected alternative was a runtime plugin
API with sandboxing — npm-installable third-party plugins. We rejected it
because a plugin that can register a Sequelize hook can corrupt anyone's data,
and sandboxing Node is a career, not a feature. So plugins are compile-time:
they live in-tree, get globbed by `PluginManager`, and are trusted code.

The wart: `PluginValueMap` in `PluginManager.ts` has an `any`-shaped escape
hatch for email templates because heterogeneous generic props defeated us.
Every time I see that comment I wince slightly.

## Evidence to go read

- `server/index.ts` — boot, throng, the collaboration/WEB_CONCURRENCY clamp.
- `server/services/index.ts` and siblings — the six services.
- `server/models/Event.ts` — note the queue import and the hooks.
- `server/queues/processors/WebsocketsProcessor.ts` — the biggest event
  consumer; a giant switch over event names that fans out to socket.io rooms.
- `server/utils/PluginManager.ts` — `Hook` enum, `PluginPriority`, glob loading.
- `plugins/slack/`, `plugins/search-postgres/` — one integration plugin, one
  infrastructure plugin. Compare their `plugin.json`/server registration.

## An incident this design caused

We once shipped a processor that threw on a malformed event payload. Bull
retried it, it threw again, and with enough poisoned events the worker queue
backed up until *websocket broadcasts* — which share the queue infrastructure —
lagged by minutes. Users saw documents "not updating" for other viewers and
filed it as a collaboration bug; it was actually a notifications bug three
subsystems away. That's the tax on an event spine: failure domains blur. The
fixes were boring and correct: defensive parsing in processors, per-processor
error isolation in `server/queues/processors/index.ts`'s dispatch, and queue
depth metrics (see 06-observability). The lesson stuck: **anything that
consumes the spine must fail alone.**

## Before you move on

1. Trace a `documents.update` call end-to-end: which service handles the HTTP,
   which table row records it, which processors react, and how does another
   browser tab find out?
2. Why does running collaboration without `REDIS_COLLABORATION_URL` force a
   single web process, and what specifically breaks if you remove the clamp?
3. You need to add a "send a Discord message when a document is published"
   feature. Which directories do you touch, and — more importantly — which
   ones do you *not* touch?
