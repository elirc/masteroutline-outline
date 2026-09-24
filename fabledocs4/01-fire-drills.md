# 01 — Fire Drills: Six Incidents, You're On-Call

Format per drill: **Page** (what the alert says), **Symptoms** (what you'd
see if you looked), then three escalating **Hints**, then the **Resolution
path** and the **Root-cause writeup** you'd file afterwards. Write your
hypothesis and first three diagnostic moves before opening Hint 1. Timebox:
45 minutes per drill before reading the resolution.

These are simulations, but every one is built on a real seam documented in
fabledocs2/3 — the mechanisms are all in this codebase, and the diagnostic
tools are the observability stack from fabledocs2 doc 06.

---

## Drill 1: "Documents aren't updating for other people"

**Page:** No page. Support tickets: users report that edits by teammates
"don't show up until I refresh." Started ~40 minutes ago. API latency normal.

**Symptoms if you look:** Collaboration works — two users *in the editor*
see each other's cursors fine. But sidebar titles, document lists, and
notification badges lag by minutes. Bull queue depth metric is climbing
steadily. Worker service CPU is normal.

<details><summary>Hint 1</summary>
Live cursors are Hocuspocus (collaboration service, direct websocket).
Sidebar/list updates are the <em>other</em> realtime path: Event row →
Bull queue → <code>WebsocketsProcessor</code> → socket.io. One path is
healthy, one is backed up. What do they not share?
</details>

<details><summary>Hint 2</summary>
Queue depth climbing + normal worker CPU = workers busy-waiting or retrying,
not crunching. Check Bull's failed/retry counts and the worker logs with
category <code>processor</code>. Look for the same job id appearing
repeatedly.
</details>

<details><summary>Hint 3</summary>
Find the event name on the retrying jobs. One processor is throwing on a
malformed payload — Bull retries it, and while retry slots are consumed,
everything behind it waits. The websocket broadcasts share worker capacity
with the poisoned consumer (ADR-006's "blurred failure domains").
</details>

**Resolution path:** Identify the poisoned event(s) from failed-job payloads;
if the processor bug is trivial, hotfix and deploy workers; if not, drop or
quarantine the specific failed jobs so the queue drains, then fix properly.
Verify queue depth falls and a test edit propagates to a second browser.

**Root-cause writeup:** A processor threw on an unexpected payload shape;
at-least-once retries amplified one bad event into queue-wide lag. Users
experienced it as "collaboration broken" — three subsystems away from the
actual bug. Preventions: defensive parsing in processors, per-processor error
isolation, alert on queue depth *and* failed-job rate, not just latency.

---

## Drill 2: The 3 a.m. pool drain

**Page:** API p99 latency critical. Error rate rising: timeouts on random,
unrelated endpoints — `collections.list`, `users.info`, anything.

**Symptoms:** Traces show requests spending their time in *pool acquire*,
not in queries. Active Postgres connections pinned at the pool max. No slow
query in `pg_stat_activity`'s top talkers — but a handful of connections
have been in `idle in transaction` or waiting on a lock for minutes.
`database`-category logs show repeated `PersistenceExtension` lines for one
document id.

<details><summary>Hint 1</summary>
Pool starvation, not slow queries. Something is holding connections open.
What in this codebase holds a transaction open for a long time on a
<em>document row</em>?
</details>

<details><summary>Hint 2</summary>
Two paths lock the same row: <code>documentCollaborativeUpdater</code>
(persisting a Y.Doc, transactional) and
<code>PersistenceExtension.onLoadDocument</code>'s lazy-conversion branch
(<code>FOR UPDATE</code>). What makes the persist slow enough to matter?
A very large state blob. What piles up behind it? Every open of that
document.
</details>

<details><summary>Hint 3</summary>
This is the lock convoy from fabledocs2 doc 03. Check the document's state
blob size. The <code>SET LOCAL lock_timeout = '15s'</code> guard bounds each
waiter, but under enough retries and openers, waiters still occupy pool
slots. Find the document, and you find the incident.
</details>

**Resolution path:** Identify the hot document id from logs; terminate the
blocking backend if one is wedged (`pg_terminate_backend`); the lock_timeout
means waiters self-clear once the head of the convoy dies. Longer term:
investigate *why* the blob is huge (embedded content that should be
attachments), and alert on pool utilization before saturation.

**Root-cause writeup:** One oversized document's persist held a row lock;
lazy-conversion `FOR UPDATE` readers convoyed behind it; each waiter held a
pool connection; unrelated requests starved for connections. The 15s
lock_timeout limited blast radius but didn't prevent pool occupancy. This is
the incident that put that `SET LOCAL` line in the code — you just relived it.

---

## Drill 3: Redis dies at lunch

**Page:** Multiple alerts at once: websocket connection count collapsed,
queue processing stopped, rate-limiter errors in logs. Redis is unreachable
(say: the self-hosted customer's Redis container OOMed).

**Symptoms:** API mostly *works* — reads and writes succeed. But: no realtime
updates, background jobs frozen, emails not sending, and log noise from
rate-limiter and cache paths.

<details><summary>Hint 1</summary>
Enumerate Redis's five hats (ADR-008) and predict each one's failure mode
<em>before</em> checking: socket.io adapter, Bull queues, rate limiting,
CacheHelper, collaboration sync. Which fail hard, which degrade, which fail
open?
</details>

<details><summary>Hint 2</summary>
Two questions decide the severity. (1) Are events being <em>lost</em> or
<em>delayed</em>? Event rows are in Postgres (ADR-006) — but the queue
publish after commit has nowhere to go. Does Bull buffer, error, or drop?
(2) Is rate limiting failing open or closed — are users locked out or is
abuse unmetered?
</details>

<details><summary>Hint 3</summary>
After Redis restarts, the sneaky issue: events created <em>during</em> the
outage have committed rows but never reached a queue. There is no relay to
replay them (the ADR-006 gap). What's inconsistent now? Search index,
notifications, webhooks for ~an hour of activity.
</details>

**Resolution path:** Restore Redis; verify sockets reconnect (clients retry)
and queues drain. Then the real work: identify the outage window and repair
downstream state — e.g. re-run search indexing for documents updated in the
window. Document which side effects were silently skipped.

**Root-cause writeup:** Redis is a SPOF wearing five hats; the app degraded
mostly gracefully in the moment, but the commit-to-publish gap means an
outage doesn't just delay side effects, it *drops* them, and recovery
requires manual reconciliation. (The capstone in 02 exists because of
exactly this.)

---

## Drill 4: Sentry goes quiet — too quiet

**Page:** None. During release review you notice Sentry has received almost
nothing for six hours, including a known noisy-but-reportable error that
fires a few times daily. Meanwhile a customer emails about intermittent 500s.

**Symptoms:** App healthy by metrics. Sentry dashboard near-flatline.
`http`-category logs *do* show sporadic 500s with stack traces.

<details><summary>Hint 1</summary>
500s in logs but not in Sentry: either the funnel isn't routing
(<code>onerror.ts</code> → <code>isReportable</code> → Sentry) or Sentry
ingestion is broken. Which is cheaper to check first?
</details>

<details><summary>Hint 2</summary>
Check Sentry quota/rate-limit status. Then check event volume by error id
over the last day: did something *flood* the project before the flatline?
</details>

<details><summary>Hint 3</summary>
The crying-wolf failure from fabledocs2 doc 05: a frequent, boring error got
marked <code>isReportable: true</code> (maybe in yesterday's release?), flooded
the quota, and ingestion is now rate-limited — silencing the <em>real</em>
500s. Diff yesterday's release for changes in <code>server/errors.ts</code>
or new error construction sites.
</details>

**Resolution path:** Find the mislabeled error, flip the bit, deploy; ask
Sentry support / wait for quota reset; triage the real 500s from logs in the
meantime (the logs never lied — only the routing did).

**Root-cause writeup:** One boolean with a big blast radius. Preventions:
review checklist item for `isReportable` on new errors; alert on Sentry
ingest *volume anomalies in both directions* — a flood and a flatline are
both incidents.

---

## Drill 5: "Search is showing the old version"

**Page:** None — a support escalation. A customer edited a security-sensitive
document to remove content; an hour later, searching still surfaces the
removed text in snippets. They're upset for compliance reasons. Opening the
document shows the correct, edited version.

**Symptoms:** Document renders correctly (state is right). Search snippet
shows stale text. It's been an hour — far beyond the normal
"stale for a minute" window.

<details><summary>Hint 1</summary>
Triple representation (ADR-007): the editor reads <code>state</code>; search
reads a projection. The refresh path is: debounced persist →
<code>documents.update</code> event → <code>SearchIndexProcessor</code> /
<code>DocumentUpdateTextTask</code>. Walk that chain — where are the
failure points?
</details>

<details><summary>Hint 2</summary>
An hour of staleness means the projection refresh <em>never ran</em>, not
that it's slow. Did the event fire? Check the events table for the document.
If the event row exists, did the task run and fail (worker logs, category
<code>task</code>)? If there's no event row… when does the persister
actually write? (Debounce + "is the last connection gone?")
</details>

<details><summary>Hint 3</summary>
Two classic causes: (a) the task failed silently (e.g., a crash mid-window,
or the Redis-outage event loss from Drill 3 — check the timeline), or
(b) the user still has the doc open in a tab, the debounced persist hasn't
flushed a final state, and projections update on persist, not on keystroke.
Which one fits an <em>hour</em> plus a correct render? Note the editor renders
from live Y.Doc, not from the projection — so a correct render proves nothing
about persistence.
</details>

**Resolution path:** Confirm which link broke via events table + task logs;
force the projection refresh (re-run the task / touch the document); verify
search updates. For the compliance angle: be able to state precisely when the
stale copy became unavailable.

**Root-cause writeup:** Projections are caches, and this incident is the
cost of ADR-007 made visible to a customer. Preventions: a reconciliation
job comparing `updatedAt` of state vs. projections (staleness monitor), and
honest documentation that search-visible deletion is eventually consistent —
plus a "purge now" path for compliance cases.

---

## Drill 6: The bundle that broke the browser

**Page:** After a routine PR merges, production frontend fails to load —
white screen, console error deep in the vendor bundle referencing a Node
builtin (`crypto` / `fs`).

**Symptoms:** CI was green (it wasn't, actually — check again). `yarn build`
locally reproduces a huge bundle or a hard error. The PR "just added a helper
to `shared/utils`."

<details><summary>Hint 1</summary>
ADR-002: <code>shared/</code> must run in both runtimes. What did the helper
import? Follow its import chain — one hop is enough to drag in
<code>@server/...</code> or a Node builtin.
</details>

<details><summary>Hint 2</summary>
Why didn't tests catch it? They should have — <code>shared</code> runs under
jsdom as a separate Vitest project. Check whether the new file has a test at
all, and whether the jsdom project actually <em>imports</em> the poisoned
module anywhere. Coverage gaps in <code>shared-jsdom</code> are exactly how
this slips through.
</details>

<details><summary>Hint 3</summary>
Fix is usually one of: move the helper to <code>server/utils</code> (it was
never dual-runtime), split it (pure logic in shared, Node-specific wrapper in
server), or inject the runtime dependency. Then close the gap: a test in the
jsdom project that imports the module, so the layering violation fails CI
next time.
</details>

**Resolution path:** Revert or hotfix the import; add the jsdom-side test;
consider a lint rule banning `@server/*` and Node-builtin imports under
`shared/` so the boundary is enforced mechanically, not by vigilance.

**Root-cause writeup:** The `shared/` contract is enforced by tests only
where tests exist. A boundary this important deserves static enforcement —
this drill's real lesson is "convention-enforced invariants erode; make the
machine hold the line."

---

## After-action for all drills

For each drill you run, write the three-sentence incident summary you'd post
in a status channel: what users saw, what actually broke, what prevents
recurrence. Practicing *that compression* is practicing seniority — the
diagnosis is table stakes; the communication is the differentiator.
