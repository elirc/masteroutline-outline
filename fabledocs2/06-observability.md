# 06 — Observability: Categories, Counters, Spans, and Knowing at 3 A.M.

## What it is and why it exists

Four instruments, one per question:

- **Logs** — `server/logging/Logger.ts`, Winston under a house API. Every log
  line carries a `LogCategory` (`"http"`, `"multiplayer"`, `"worker"`,
  `"database"`, ... — it's a closed union type, look at it) plus structured
  extra fields. Question answered: *what happened, in which subsystem?*
- **Metrics** — `server/logging/Metrics.ts`, StatsD via `hot-shots`.
  Counters and gauges: queue depth, websocket connections, rate-limit hits.
  Question: *how much, how fast, trending which way?*
- **Traces** — `server/logging/tracer.ts` + `tracing.ts`, dd-trace. Note the
  `@trace()` class decorator and `traceFunction()` wrapper — spans are
  declared at the code site, not configured centrally. Question: *where did
  the time go?*
- **Sentry** — `server/logging/sentry.ts`, fed only by errors the funnel
  deemed reportable (see 05). Question: *what's broken that a human must fix?*

Self-hosting shapes all of it. We cannot assume Datadog, or Sentry, or any
vendor — every sink is optional and env-gated, and the system must degrade to
"pretty console logs" on a bare Docker install. That's why `Metrics.ts`
swallows StatsD socket errors silently (a comment in the file says so — a
metrics outage must never become an app outage) and why `tracer.ts` lazily
`require()`s dd-trace only when configured, because dd-trace patches Node
internals on import and that cost is unacceptable when tracing is off.

Two structural details worth internalizing early. First,
`server/index.ts` imports `./logging/tracer` *before anything else*, with a
comment saying so — instrumentation-by-monkey-patching only sees modules
loaded after it. People have broken tracing by "organizing" those imports;
the comment is load-bearing. Second, observability extends into the
collaboration stack via Hocuspocus extensions (`LoggerExtension.ts`,
`MetricsExtension.ts` in `server/collaboration/`) — websocket sessions don't
pass through Koa middleware, so they need their own instrumentation seam.

## The hard decisions

### 1. A closed category union instead of free-form logger names

Most codebases do `getLogger("whatever.string.i.felt.like")` and end up with
four hundred logger names, none searchable. We made `LogCategory` a typed
union of ~15 values. Adding a category is a deliberate act that touches a
shared type; in exchange, "show me everything `multiplayer` logged around
14:32" is a one-filter query, and on-call runbooks can say "filter category X"
and be right forever. The rejected alternative — hierarchical logger
namespaces à la log4j — offers per-namespace level tuning we occasionally
miss, but the discipline of a closed set has been worth more than the
flexibility, and I'd make the same call again. The residual annoyance:
category granularity is uneven — `"http"` is a firehose while `"utils"`
logs twice a year — and rebalancing a closed union means touching every call
site.

### 2. Decorator-based tracing vs. auto-instrumentation everywhere

dd-trace will happily auto-instrument Koa, Sequelize, and Redis, and we do
let it. The decision was about *our* code: `@trace()` on classes like
`PersistenceExtension` and explicit `traceFunction()` wraps (see
`server/presenters/document.ts`) versus relying purely on framework spans.
Auto-only gives you "the request took 2s and spent 1.8s in... middleware."
Useless. Explicit spans give you `PersistenceExtension.onLoadDocument` as a
named unit. The rejected middle path — a build step injecting spans into
every exported function — produced trace soup in the prototype: thousands of
2ms spans hiding the one that mattered. Spans are a curated product for
future-debugging-you, not an exhaust stream. The cost, honestly carried: the
codebase's span coverage is exactly as good as our judgment about what would
matter someday, and we've been wrong in both directions.

### 3. Metrics fail open, always

`Metrics.ts` ignoring StatsD errors was contentious. The argument against:
silently dropping metrics means you can't trust a flat line — is the queue
empty, or is the metric dead? That's a real cost; a flatlined graph that
*looks* healthy is worse than no graph. The argument that won: the
alternative is a monitoring sidecar outage taking down document editing for a
self-hoster who never wanted metrics in the first place. Observability must
never be load-bearing. We accepted "can't distinguish zero from broken" and
partially mitigated it with a heartbeat metric — if *that* flatlines, the
pipe is dead, not the app quiet. It's a workaround wearing a design's
clothes, and I'm at peace with it.

## Evidence to go read

- `server/logging/Logger.ts` — the `LogCategory` union; also note it hooks
  Sentry and Metrics, making it the front door to everything else.
- `server/logging/tracer.ts` — the lazy require and its comment about
  dd-trace's import cost; `getRequestSpan` digging into `req._datadog`.
- `server/logging/Metrics.ts` — twenty lines, one philosophy.
- `server/collaboration/LoggerExtension.ts` / `MetricsExtension.ts` — the
  non-HTTP instrumentation seam.
- `server/middlewares/requestTracer.ts` and `requestContext.ts` — request ids
  and how context rides along.
- `server/queues/HealthMonitor.ts` — queue self-surveillance.

## An incident this design prevented

The lock-convoy incident from doc 03 — big document, long row lock, drained
connection pool — was *diagnosed* in about twenty minutes, and that speed was
entirely this stack. The pager fired on API latency. Traces showed requests
parked in Sequelize's pool-acquire phase, not in queries — so it wasn't a slow
query, it was pool starvation. The `database`-category logs showed
`PersistenceExtension` state-creation lines for a single document id
repeating across the window, and queue metrics were clean, which ruled out
worker interference. Three instruments, three eliminations, one culprit:
everything queued behind one document's lock. Without named spans inside the
collaboration path we'd have restarted servers (clearing the lock,
"fixing" it, learning nothing) and met the same bug the next week. The
instrumentation on that path predated the incident precisely because
`@trace()` makes adding it nearly free — the cheapest insurance this
codebase carries.

## Before you move on

1. Why must `import "./logging/tracer"` stay the first meaningful import in
   `server/index.ts`, and what silently degrades if it drifts?
2. Metrics fail open and never throw. Defend that choice, state its cost, and
   explain how the heartbeat metric mitigates (not eliminates) it.
3. API latency alerts fire at 3 a.m. Using only this stack — categories,
   spans, queue metrics, Sentry — write your first three diagnostic moves and
   what each would rule out.
