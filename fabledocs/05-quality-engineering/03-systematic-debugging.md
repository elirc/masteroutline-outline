# Systematic Debugging

The method, then five repo-specific scenarios. The method is the transferable part; the scenarios make it muscle memory.

## The method

1. **Reproduce** — minimal, deterministic. If you can't reproduce, your job is *observability first*, not fixing.
2. **Narrow** — binary-search the system: which side of a boundary is wrong? (Client cache vs API response? API vs DB? Sync path vs async path?) Outline's boundaries make great cut points: ApiClient response, `ctx.input` after validate, the committed row, the emitted event, the processor's job.
3. **Hypothesize** — one falsifiable statement at a time.
4. **Test cheaply** — a log line, a single SQL query, one curl — before any code change.
5. **Fix the root cause** — not the symptom's location.
6. **Add regression coverage** — the test you wish had existed.

Actual tools here: browser devtools + MobX state inspection; `node --inspect` (already in the dev script, [package.json L13](../../package.json#L13)); Sequelize query logging; Bull dashboard via the **admin** service (bull-board is a dependency — run services with `admin` and inspect queues); `Logger` categories ([server/logging/Logger.ts](../../server/logging/Logger.ts)); `/_health` ([server/index.ts L159–177](../../server/index.ts#L159-L177)); test factories for reproduction in a test instead of clicking.

---

## Scenario 1: "My teammate renamed a document but I still see the old title until I refresh"

Reproduction: two browsers, same team, rename in one.
First question: did the second browser *receive* the websocket event (transport problem) or receive-and-ignore it (client merge problem)?
Narrowing path:
1. Second browser devtools → WS frames on `/realtime`. Event present?
2. No event → is the socket in the right rooms? Check the `join` handshake ([WebsocketProvider L126–128](../../app/components/WebsocketProvider.tsx#L126-L128)); then server side: did the worker enqueue a websockets job ([worker.ts L63–66](../../server/services/worker.ts#L63-L66))? Bull board → `websockets` queue.
3. Event arrived → breakpoint in the `documents.update` handler ([WebsocketProvider L253–262](../../app/components/WebsocketProvider.tsx#L253-L262)); does `documents.add(event)` run? Does the title in store change but the component not re-render (observer wrapping missing)?
Useful probes: `localStorage.debug = '*'` style socket.io client logging; MobX `spy`.
Likely root causes: socket never joined the document/collection room; event payload missing `title`; component reading a non-observable copy.
Regression test: store-level test that `documents.add` with a partial updates `title`.
Senior lesson: real-time bugs are almost always *room membership* or *merge* bugs; instrument the seam, don't shotgun.
Interview version: narrate the transport-vs-merge split first — interviewers reward the explicit binary search.

## Scenario 2: "A user reports 403 on a document they can clearly see in their sidebar"

Reproduction: get ids (user, doc); in a test, rebuild the membership shape with factories.
First question: is the sidebar wrong (stale client policy) or the API wrong (policy evaluation)?
Narrowing path:
1. Curl `documents.info` as the user (API key) → 403 confirms server-side.
2. Which action failed? `authorize(user, "read", document)` — replicate in a test: build user/collection/doc, call `can(user, "read", doc)`.
3. Walk the [read policy](../../server/policies/document.ts#L21-L34) branch by branch: team match? membership preloaded? draft-author? collection readable?
4. Suspect **preloading**: was the document loaded without `userId`, so `memberships` is empty (or the invariant at [L283–290](../../server/policies/document.ts#L275-L308) throws in dev but prod data path differs)?
Useful probes: log `document.memberships?.length`, `document.collection?.id` right before authorize.
Likely root causes: call site loading the doc without the membership scope; group membership not synced ([groupsSyncer](../../server/commands/groupsSyncer.ts)); sidebar showing via a *collection* grant that was since revoked (client cache staleness).
Regression test: policy test with the exact membership topology.
Senior lesson: authorization bugs are usually *data-loading* bugs. The policy is pure; the inputs were wrong.
Interview version: "I check what the policy *saw*, not what the rule *says*."

## Scenario 3: "Notification emails stopped going out, but the app works fine"

First question: are events reaching the worker at all, or dying between processor and email?
Narrowing path (follow the pipeline from [Flow 6](../../fabledocs/01-codebase-cartography/05-key-flows.md)):
1. Bull board: `globalEvents` queue depth normal? Failed jobs?
2. `processorEvents` queue: NotificationsProcessor jobs failing? Read the error — worker logs include event context ([worker.ts L148–153](../../server/services/worker.ts#L140-L155)).
3. Jobs green but no email → Notification rows created? (DB check.) Then the email stage: `EmailsProcessor`/`EmailTask` failures, SMTP config.
4. Nothing anywhere → is a **worker service running at all**? (`SERVICES` env missing `worker` is the classic self-hosting failure.)
Useful probes: queue depths over time (HealthMonitor exists — [server/queues/HealthMonitor.ts](../../server/queues/HealthMonitor.ts)); grep logs for the task name.
Likely root causes: worker not running; SMTP creds rotated; a poisoned job exhausting retries (check `onFailed`); user-level: recipient muted/unsubscribed (working as intended!).
Regression: alert on queue depth + failed-job count, not a code test.
Senior lesson: async failures need *stage-by-stage evidence*; also "bug" reports are sometimes correct behavior (mutes) — verify intent before code.
Interview version: perfect for "tell me about debugging a distributed issue."

## Scenario 4: "documents.search is slow for one big team"

First question: slow in the database or around it?
Narrowing path:
1. Timings: the endpoint records search duration in SearchQuery rows — query those first (data you already have!).
2. `EXPLAIN ANALYZE` the provider's SQL (from [plugins/search-postgres](../../plugins/search-postgres)) with that team's parameters.
3. Check the permission pre-computation: `user.collectionIds()` cost for users with huge collection counts; then tsvector index usage (seq scan = missing/unused index).
4. Rule out the non-DB parts: snippet generation, ranking, presenter N+1s (does presenting results trigger per-doc queries?).
Useful probes: Sequelize logging with timestamps; Postgres `pg_stat_statements`.
Likely root causes: seq scan from a query shape the index doesn't serve; oversized `limit`; popularity-boost join.
Regression: a perf assertion is brittle — instead commit the `EXPLAIN` expectation in a comment and add an index migration with before/after numbers in the PR.
Senior lesson: measure where the time *is* before touching code; the app already collects timing telemetry — use product data as debugging data.
Interview version: the follow-up will be "how do you fix an N+1?" — have the batch-presenter example ready ([presentDocuments L147–176](../../server/presenters/document.ts#L147-L176)).

## Scenario 5: "After deploying, some clients get errors calling the API" (version skew)

Reproduction: hard — happens only during the deploy window with old tabs open.
First question: request-shape skew (old client, new validation) or response-shape skew (new presenter, old client)?
Narrowing path:
1. Error logs: `ValidationError` spikes right after deploy → new/stricter Zod schema rejecting old clients' payloads.
2. Client-side Sentry: undefined-field crashes → response shape changed.
3. Check what the codebase already does about skew: client sends `x-client-version` and editor version headers; server can respond `UpdateRequiredError` → forced refresh path in ApiClient.
Likely root causes: renamed/removed request field without accepting the old one for a deploy cycle; websocket payload changes hitting old tabs (they hold connections for hours!).
Fix pattern: **expand → migrate → contract** for contracts, not just DB schemas: accept both → ship clients → remove old.
Regression: contract tests on presenters (snapshots); a checklist item for schema changes.
Senior lesson: deploys are not atomic across browser tabs; every wire change must be one-version-compatible.
Interview version: this is the "how do you do zero-downtime deploys?" answer from the trenches.

---

Drill: pick Scenario 2 or 3 and actually *perform* it — inject the bug locally (revoke a membership in psql / stop the worker), then walk your own narrowing path with real evidence. Time it. Under 20 minutes to root cause = interview-ready for debugging rounds ([08-interview-prep/05](../08-interview-prep/05-debugging-and-code-review-rounds.md) has the timed versions).
