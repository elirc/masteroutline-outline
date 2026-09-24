# 02 — Capstone: Close the Commit-to-Publish Gap (Transactional Outbox Relay)

## The problem you're solving

ADR-006 named it plainly: mutations write an `Event` row transactionally
(good), then publish to Bull *after commit, in-process* (the gap). A crash —
or a Redis outage, see Drill 3 — between commit and publish leaves a
committed event that no processor ever consumes. The events table is an
outbox in shape without the relay that makes outboxes safe.

You're going to build the relay. This is the single most instructive
exercise this codebase offers: it touches the data layer, the queue layer,
delivery semantics, idempotency, and operational rollout — and it's a
pattern you'll reuse for the rest of your career.

**Prerequisites:** fabledocs2 docs 01/03, ADR-005/006, Drill 3. A running
dev environment.

## Stage 0 — Design doc (half a day, do not skip)

One page answering:

1. **How does the relay know what's unpublished?** You need a marker —
   nullable `publishedAt` (or `processedAt`) column on `events`, set when
   enqueue succeeds. Write the migration plan (`server/migrations/`,
   `yarn db:create-migration`). What's the default for the millions of
   *existing* rows, and why must it not be NULL for them? (If old rows look
   unpublished, your relay replays history — see Stage 3.)
2. **Where does the relay run?** You have a `cron` service and a scheduled-
   task pattern (look at how cron tasks are registered; `server/services/cron.ts`,
   `server/routes/api/cron/`). Every N seconds: claim a batch of unpublished
   events older than a grace window, enqueue, mark published.
3. **What are the semantics now?** Normal path: publish-then-mark means a
   crash after publish but before mark = republish = **at-least-once into
   the queue**. You are *choosing* duplicates over loss. Say so explicitly,
   and say why the reverse order (mark-then-publish) silently reintroduces
   loss.
4. **Concurrency:** two cron ticks or two cron pods must not both claim the
   same rows. `SELECT ... FOR UPDATE SKIP LOCKED` is the canonical answer —
   you'll have practiced it in the SQL lab (04). Sketch the exact query.
5. **The grace window:** the in-process publish still exists and is still
   the fast path. The relay only sweeps events older than, say, 30 seconds —
   old enough that the fast path clearly failed. Why keep the fast path at
   all? (Latency: sidebar updates shouldn't wait for a cron tick.)

Grade your design against this checklist before coding. If your design
marks-then-publishes, or lets two relays claim one row, or replays all of
history on first deploy — revise.

## Stage 1 — The migration and the marker (half a day)

- Migration adding `publishedAt` (timestamptz, nullable, indexed
  partial-where-null — justify the partial index).
- Backfill strategy for existing rows: set `publishedAt = createdAt` in the
  migration (or batched, if you decide the table's too big for one UPDATE —
  defend your choice; `CleanupOldEventsTask` tells you the table can be
  large).
- Update the post-commit publish path (find where `Event` reaches
  `globalEventQueue` — start from the queue import in
  `server/models/Event.ts`) to set `publishedAt` after successful enqueue.
  Decide: is that update its own tiny write, and what happens if *it* fails
  after a successful enqueue? (Duplicate later, by design. Fine. Write the
  comment saying so.)

## Stage 2 — The relay task (1 day)

- A new scheduled task (follow the naming and structure of existing
  `server/queues/tasks/Cleanup*Task.ts` files; register it the way the
  plugin/task system expects — check `PluginManager` `Hook.Task` and how
  cron schedules the cleanup tasks).
- Core loop, in a transaction:
  `SELECT id, ... FROM events WHERE "publishedAt" IS NULL AND "createdAt" < now() - interval '30 seconds' ORDER BY "createdAt" LIMIT 100 FOR UPDATE SKIP LOCKED`
  → enqueue each → `UPDATE events SET "publishedAt" = now() WHERE id IN (...)`.
- Metrics (fabledocs2 doc 06 conventions): counter for relayed events —
  **this metric is the whole point operationally**: nonzero = the fast path
  failed = something to investigate. Add a gauge for unpublished backlog age.
- Logging under an appropriate `LogCategory`.

## Stage 3 — Consumer idempotency audit (half a day, the honest part)

At-least-once into the queue means duplicates reach processors. Pick the
three highest-blast-radius consumers and audit them for a duplicated event:

- `WebsocketsProcessor` — duplicate broadcast. Harm? (Basically none —
  clients re-render. Document why it's safe.)
- `NotificationsProcessor` / `EmailsProcessor` path — duplicate email. Harm?
  (Real. What's the dedupe key? Does a `Notification` row already exist you
  can upsert on?)
- `RevisionsProcessor` — duplicate revision snapshot. Harm? (Check: it's
  already debounced/deduped by design — verify against
  `DebounceProcessor.ts` and say *why* it's safe or isn't.)

Deliverable: a short table — consumer / duplicate effect / existing guard /
needed guard. Implement one missing guard.

## Stage 4 — Tests (half a day)

Per ADR-010 doctrine — real Postgres, factories:

1. Event created with a simulated enqueue failure (mock the queue seam only)
   → row has `publishedAt` NULL → relay run → job enqueued → `publishedAt`
   set.
2. Relay respects the grace window (fresh unpublished events untouched).
3. Two concurrent relay runs don't double-claim (this one's tricky — two
   transactions in one test; the SQL lab shows you the shape).
4. Crash-after-publish simulation → duplicate delivered → your Stage-3
   guarded consumer stays idempotent.

## Acceptance criteria

- [ ] Design doc states the delivery semantics and the ordering rationale.
- [ ] Migration is deploy-safe on a large existing table (no replay storm).
- [ ] Relay uses `SKIP LOCKED`; concurrent runs are safe.
- [ ] Fast path preserved; relay is sweep-only.
- [ ] Relayed-event metric + backlog-age gauge exist.
- [ ] Idempotency audit table written; one guard implemented.
- [ ] All four tests pass; `yarn test:server` green.

## Interview payoff

After this, ADR-006's follow-up question — "how would you add the relay
without double-delivering existing events?" — is no longer hypothetical. Your
answer starts with "I've built exactly this," and the backfill-to-createdAt
detail *is* the answer to that question. Be ready to explain why you chose
at-least-once over exactly-once (spoiler: exactly-once into Redis isn't on
the menu; idempotent consumers are how grown-ups spell it).
