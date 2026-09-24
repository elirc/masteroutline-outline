# ADR-008: Redis as the Sole Horizontal-Scale Substrate

**Status:** Active

## Context

Once the web/websocket services scale past one process (ADR-001), every piece
of cross-process coordination needs a home: which server holds a user's
socket when an event must reach them, whether two processes are about to do
the same expensive work, how rate limits stay consistent across pods, where
hot data is cached. Constraint, as always: self-hosters tolerate exactly one
piece of infrastructure beyond Postgres.

## Decision

Redis is the answer to all of it — one dependency, many roles:

- **Websocket fan-out:** socket.io with `socket.io-redis` adapter
  (`server/services/websockets.ts`), so a `WebsocketsProcessor` broadcast
  from any worker reaches sockets held by any web process, via rooms keyed to
  users/collections/documents.
- **Queues:** Bull persists jobs in Redis (ADR-006).
- **Distributed locks + cache:** `server/utils/CacheHelper.ts` implements
  get-or-compute with a distributed lock to prevent cache stampede
  (multiple processes computing the same miss).
- **Rate limiting:** `rate-limiter-flexible` with Redis storage
  (`server/middlewares/rateLimiter.ts`), keyed on user id when a token
  resolves to one (cached via `RateLimiter.getCachedUserIdForToken`), IP
  otherwise — so an office behind one NAT doesn't share a bucket, and limits
  hold across processes.
- **Collaboration scale-out:** `REDIS_COLLABORATION_URL` enables multi-node
  collaboration; without it, Y.Doc state is process-local and the process
  count clamps to 1 (ADR-001's scar).
- **Namespacing:** `server/utils/RedisPrefixHelper.ts` keeps roles from
  colliding in one Redis.

## Alternatives Considered

1. **Postgres for everything** (LISTEN/NOTIFY fan-out, advisory locks,
   `FOR UPDATE SKIP LOCKED` queues). Viable and one-fewer-dependency, but
   LISTEN/NOTIFY payload limits and connection-pool pressure (every web
   process holding listen connections, on top of ADR-007's lock-heavy
   workload) made Postgres the resource to protect, not load further.
2. **Dedicated brokers per concern** (RabbitMQ for queues, memcached for
   cache...). Rejected: each is individually better; collectively they're an
   operations tax self-hosters won't pay.
3. **Sticky sessions instead of pub/sub fan-out.** Rejected: pinning users to
   processes fails at deploys and rebalances, and cross-user broadcasts
   (someone else edited this doc) still need cross-process reach.

## Consequences

**Positive:**
- One `REDIS_URL` and the system scales horizontally; deployment stays
  self-hoster-simple.
- Stampede-safe caching and consistent limits come from shared helpers, not
  per-callsite invention.

**Negative:**
- Redis is a single point of failure wearing five hats: an outage
  simultaneously degrades websockets, queues, rate limiting, cache, and
  collaboration sync — failure modes couple across concerns that would be
  isolated with dedicated infrastructure.
- Bull jobs are the only *durable* data in Redis, but operators treat Redis
  as ephemeral cache; a flush loses queued work (retries mask small losses,
  not a flush).
- Every Redis role has a "works without it, worse" degradation path to
  support (in-process everything), doubling the test surface — the
  single-process clamp being the sharpest example.

## Evidence

- `server/services/websockets.ts` — `createAdapter` from `socket.io-redis`;
  room joins gated by `can` (ADR-004).
- `server/storage/redis.ts` — the shared client.
- `server/utils/CacheHelper.ts` — documented distributed-lock get-or-compute.
- `server/middlewares/rateLimiter.ts` — `getRateLimiterIdentifier`, token→
  user caching.
- `server/utils/RedisPrefixHelper.ts` — key namespacing.
- `server/index.ts` — `REDIS_COLLABORATION_URL` branch.
