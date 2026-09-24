# Side Effects, Async, and Reliability

## The side-effect map

Every mutation in Outline potentially triggers work beyond its own row writes. The complete inventory, by mechanism:

| Side effect | Trigger mechanism | Code home |
| --- | --- | --- |
| Audit log | `Event.create*` in-transaction | [Event.ts](../../server/models/Event.ts) |
| Websocket pushes to clients | event → dedicated `websockets` queue | [worker.ts L63–66](../../server/services/worker.ts#L62-L67), [WebsocketsProcessor](../../server/queues/processors/WebsocketsProcessor.ts) |
| Revisions | event (debounced) → processor → command | [RevisionsProcessor](../../server/queues/processors/RevisionsProcessor.ts) |
| Notifications → emails | event → processor → per-recipient tasks → email send | [NotificationsProcessor](../../server/queues/processors/NotificationsProcessor.ts), [EmailsProcessor](../../server/queues/processors/EmailsProcessor.ts) |
| Outgoing webhooks | event → webhooks plugin processors | [plugins/webhooks/server](../../plugins/webhooks/server) |
| Search index (external providers) | event → processor | [SearchIndexProcessor](../../server/queues/processors/SearchIndexProcessor.ts#L21-L46) |
| Backlinks graph | event → processor | [BacklinksProcessor](../../server/queues/processors/BacklinksProcessor.ts) |
| File imports/exports | explicit tasks with FileOperation state rows | [ExportTask family](../../server/queues/tasks/ExportMarkdownZipTask.ts), [ImportsProcessor](../../server/queues/processors/ImportsProcessor.ts) |
| Uploads to S3 (avatars, attachments-from-URL) | explicit tasks | [UploadUserAvatarTask](../../server/queues/tasks/UploadUserAvatarTask.ts), [UploadAttachmentFromUrlTask](../../server/queues/tasks/UploadAttachmentFromUrlTask.ts) |
| Housekeeping (purges, rollups, reminders) | cron service → CronTask subclasses | [CronTask](../../server/queues/tasks/base/CronTask.ts), Cleanup* tasks |
| External API calls (Slack, GitHub…) | integration processors/tasks in plugins | [plugins/*](../../plugins) |

The architectural rule this table encodes: **the request path does synchronous DB writes only**; everything else rides the event bus or an explicit task. When you find a side effect *in* a request path (an HTTP call to a third party, say), that's a review flag — check timeout, failure handling, and whether it holds the transaction open.

## Reliability vocabulary, grounded

- **Idempotency** — safe to run twice. Required *everywhere* here because Bull retries jobs (default 5 attempts, exponential backoff — [queues/index.ts](../../server/queues/index.ts), [BaseTask.options](../../server/queues/tasks/base/BaseTask.ts#L48-L60)) and the global fan-out re-enqueues on partial failure ([worker.ts L80–92](../../server/services/worker.ts#L80-L93)). Study how each processor achieves it: RevisionsProcessor by content-equality guard; notification tasks by checking for existing Notification rows (verify per task); cleanup tasks naturally (deleting deleted things).
- **Retries with backoff** — exponential, capped attempts; after the last, `onFailed` runs ([worker.ts L142–146](../../server/services/worker.ts#L140-L155)) — the hook for compensation or alerting. A task with an empty `onFailed` fails *silently* after retries: acceptable for popularity rollups, not for exports a user is waiting on (exports mark their FileOperation failed — see [ErrorTimedOutFileOperationsTask](../../server/queues/tasks/ErrorTimedOutFileOperationsTask.ts) sweeping up even the crashed ones — that's **failure visibility** done right: a *watchdog for the watchdog*).
- **Outbox** — the pattern of writing "message to send" in the same transaction as the data, with a relay publishing it later. Outline is *half*-outbox: events are transactional rows, but publish is a post-commit push with no relay re-reading missed rows ([Event.ts L82–97](../../server/models/Event.ts#L82-L97)). Gap and fix discussed in [Pattern 8](05-pattern-catalog.md) and the [risk register](../09-reference/risk-register.md).
- **Backpressure / flow control** — queue depths are the buffer; [HealthMonitor](../../server/queues/HealthMonitor.ts) watches queues; priorities ([TaskPriority](../../server/queues/tasks/base/BaseTask.ts#L4-L9)) keep urgent work ahead when the buffer fills. Concurrency knobs: `WORKER_CONCURRENCY_EVENTS` / `WORKER_CONCURRENCY_TASKS` ([worker.ts](../../server/services/worker.ts#L22-L28)).
- **Timeouts** — websocket queue jobs get 10s ([queues/index.ts](../../server/queues/index.ts)); HTTP server has `REQUEST_TIMEOUT` ([server/index.ts L221](../../server/index.ts#L220-L222)); imports/exports get watchdog tasks (ErrorTimedOut*). Every external wait in a system needs an owner-of-the-timeout — good audit exercise.
- **Compensation** — undoing a half-done multi-step job. Mostly avoided here by making steps idempotent and re-runnable rather than compensating; FileOperation state machines mark failure instead of rolling back.

## Flags and investigate-items (practice judgment)

- *Investigate:* `DebounceProcessor` semantics — what delay, what happens to events during a worker restart mid-debounce? Read [DebounceProcessor.ts](../../server/queues/processors/DebounceProcessor.ts) and write the answer down.
- *Possible risk:* event fan-out failure granularity — if 1 of N processors' `add` fails, the global job rethrows and *all* processors get re-enqueued on retry ([worker.ts L80–92](../../server/services/worker.ts#L80-L93)); consumers must tolerate duplicates (most do; verify webhooks dedupe before assuming).
- *Possible risk:* task props are raw JSON on Redis — schema drift between deploys (a job enqueued by old code, consumed by new) is unversioned. Low frequency, worth knowing exists.

## Interview angle

1. *"How do you send emails without slowing requests?"* — the event → processor → task → email pipeline with retries and priorities; sketch [Flow 6](../01-codebase-cartography/05-key-flows.md).
2. *"What happens when a background job fails?"* — backoff → attempts exhausted → `onFailed` → watchdog sweepers for user-visible operations. Naming the *watchdog for the watchdog* (ErrorTimedOut* tasks) is a memorable senior beat.
3. *"Exactly-once processing?"* — doesn't exist; at-least-once + idempotent consumers is the honest answer, with this repo's guards as examples.
4. *"How would you add webhooks to a product?"* — subscription model + event bus tap + delivery rows with retry state ([plugins/webhooks](../../plugins/webhooks/server)).

## Drill

Pick `documents.delete`. Enumerate: (1) every processor/task that fires (grep `documents.delete` in server/queues and plugins), (2) for each, what happens if it runs twice, (3) what a user observes if each fails permanently. Write the 3-column table. Self-grade — Basic: found the consumers. Solid: correct idempotency verdicts. Strong: identified the one consumer whose permanent failure a user would actually notice, and what alert would catch it.
