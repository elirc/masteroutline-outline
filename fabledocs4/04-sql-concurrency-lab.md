# 04 — SQL & Concurrency Lab: Two psql Sessions and the Truth

Everything in ADR-005/007 about locks, and the entire Drill 2 incident,
comes down to Postgres behaviors you can reproduce on your dev database in
an afternoon with two terminal windows. Do these in order; each experiment
sets up intuition the next one uses. Keep both sessions visible side by side.

Setup: `psql` twice into your dev database (call them **S1** and **S2**).
Work on a scratch table so you can't hurt your dev data:

```sql
CREATE TABLE lab_docs (id int PRIMARY KEY, title text, locked_by text);
INSERT INTO lab_docs SELECT g, 'doc ' || g, NULL FROM generate_series(1, 10) g;
```

---

## Experiment 1: What FOR UPDATE actually blocks

- S1: `BEGIN; SELECT * FROM lab_docs WHERE id = 1 FOR UPDATE;` (leave open)
- S2: `SELECT * FROM lab_docs WHERE id = 1;` → **returns instantly.** Plain
  reads don't block. First lesson: `FOR UPDATE` is writer-vs-writer, not
  reader-vs-writer (MVCC).
- S2: `SELECT * FROM lab_docs WHERE id = 1 FOR UPDATE;` → **hangs.** This is
  exactly `PersistenceExtension.onLoadDocument` queuing behind a persist.
- S1: `COMMIT;` → S2 unblocks and sees committed data.

**Connect to codebase:** re-read the fast path in `PersistenceExtension` —
it checks `state` *without* a lock first, precisely to avoid joining this
queue when it doesn't have to. Now you know what it's avoiding.

## Experiment 2: lock_timeout, the 15-second tombstone

- S1: `BEGIN; SELECT * FROM lab_docs WHERE id = 1 FOR UPDATE;`
- S2:
  ```sql
  BEGIN;
  SET LOCAL lock_timeout = '2s';
  SELECT * FROM lab_docs WHERE id = 1 FOR UPDATE;
  ```
  → after 2 seconds: `ERROR: canceling statement due to lock timeout`, and
  S2's transaction is aborted.
- Note `SET LOCAL` scope: it dies with the transaction — run
  `SHOW lock_timeout;` in S2 after `ROLLBACK;` to confirm it's back to
  default. That's why `documentCollaborativeUpdater.ts` uses `SET LOCAL`,
  not `SET`: no leakage onto the pooled connection's next borrower. Say that
  last sentence out loud; it's a great interview detail.

## Experiment 3: Build the convoy from Drill 2

- S1: `BEGIN; SELECT * FROM lab_docs WHERE id = 1 FOR UPDATE;` — this is the
  slow persist of the oversized document.
- Open S3 and S4 (yes, two more terminals): both
  `SELECT * FROM lab_docs WHERE id = 1 FOR UPDATE;` — these are document
  opens lazily converting. Both hang.
- In a fifth session, look at the crime scene:
  ```sql
  SELECT pid, state, wait_event_type, wait_event, query
  FROM pg_stat_activity WHERE query ILIKE '%lab_docs%';
  ```
  See the waiters (`wait_event_type = 'Lock'`). Now:
  ```sql
  SELECT blocked.pid AS blocked_pid, blocking.pid AS blocking_pid
  FROM pg_stat_activity blocked
  JOIN pg_stat_activity blocking
    ON blocking.pid = ANY(pg_blocking_pids(blocked.pid));
  ```
  `pg_blocking_pids` is the single most useful function in an incident —
  it names the head of the convoy.
- Each hung session is a *held pool connection*. Count your app's pool size
  (check Sequelize config in `server/storage/database.ts`); imagine each
  waiter occupying a slot, and API requests arriving to an empty pool.
  That's the whole incident, reproduced for free.
- Clean up: `SELECT pg_terminate_backend(<blocking_pid>);` from the fifth
  session — and watch the convoy drain. That's your Drill-2 resolution
  command, now rehearsed.

## Experiment 4: SKIP LOCKED — the relay's trick

(This is the mechanism your outbox capstone depends on.)

- S1: `BEGIN; SELECT * FROM lab_docs WHERE id <= 3 FOR UPDATE SKIP LOCKED;`
  → rows 1–3.
- S2: `BEGIN; SELECT * FROM lab_docs WHERE id <= 6 FOR UPDATE SKIP LOCKED;`
  → rows 4–6 **instantly** — no hang, no overlap. Two workers, disjoint
  batches, zero coordination code. This one-line pattern is how the
  capstone's concurrent relay runs stay safe, and how every
  Postgres-as-job-queue system works.
- Commit both; run S2's query again → now it can see 1–6.

## Experiment 5: The half-commit the transaction middleware prevents

Simulate the pre-ADR-005 bug class:

- S1 (autocommit, no BEGIN): `UPDATE lab_docs SET title = 'edited' WHERE id = 7;`
  then *pretend the process crashed here* — the "event insert" never runs.
  Row 7 is edited; no event exists. Downstream (websockets, search, webhooks)
  will never know. Nothing in the database is "wrong" — that's what makes it
  silent.
- Now the fixed version: `BEGIN;` update + insert a fake event row (add
  `CREATE TABLE lab_events (id serial, doc_id int);` if you like) +
  `COMMIT;` — kill it anywhere before COMMIT and *neither* happens.
  Atomicity isn't about corruption; it's about **downstream consumers'
  worldview staying consistent**. That's the sentence that justifies passing
  `transaction` through every command signature in `server/commands/`.

## Experiment 6: What the ORM hides

Turn on statement logging for your session pair
(`SET log_statement = 'all'` needs superuser; alternative: run one real API
mutation with Sequelize `logging: console.log` temporarily enabled in
`server/storage/database.ts`, or set the env debug flag if one exists) and
watch what `documents.update` actually emits: the `BEGIN`, the scoped
`SELECT`, the `UPDATE`, the `INSERT INTO events`, the `COMMIT`. Count the
round trips. The ORM is convenient; it is not magic; and you should see the
`SET LOCAL lock_timeout` appear verbatim when the collaborative updater runs.

Cleanup: `DROP TABLE lab_docs; DROP TABLE IF EXISTS lab_events;`

---

## Exit questions

1. Why does `PersistenceExtension` read without a lock before reading with
   one, and what specific queue does the fast path bypass?
2. `SET` vs `SET LOCAL` on a pooled connection — what's the failure mode of
   the wrong one, and who suffers it? (Not the current request…)
3. Your outbox relay runs on two cron pods simultaneously. Write, from
   memory, the SELECT that makes this safe, and explain what each of
   `FOR UPDATE` and `SKIP LOCKED` contributes independently.
4. In an incident, what does `pg_blocking_pids` give you that
   `pg_stat_activity.state` alone doesn't?
