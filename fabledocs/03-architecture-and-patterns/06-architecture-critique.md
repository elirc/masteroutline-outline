# Architecture Critique

The "what do you think of this codebase?" exercise, done properly: strengths first (understand before judging), then risks with evidence, then a prioritized change list *as if you owned it for 3 months*. This file doubles as system-design interview material — cross-linked from [08-interview-prep/04](../08-interview-prep/04-system-design-from-this-repo.md).

Confidence labels: **confirmed** (traced in code), **hypothesis** (pattern suggests it; needs measurement or deeper reading).

## Strongest design choices (steal these)

1. **Policy engine with client serialization** (confirmed). One authorization brain, rendered by both enforcement and UI ([cancan.ts](../../server/policies/cancan.ts), [usePolicy](../../app/hooks/usePolicy.ts)). Eliminates the most common permission-drift bug class entirely.
2. **Uniform route pipeline** (confirmed). `auth → validate → rateLimit → transaction → authorize → command → present` readable in every route signature ([documents.ts](../../server/routes/api/documents/documents.ts)). Predictability is a feature: reviewers see deviations instantly.
3. **Transaction-coupled events with post-commit publish** (confirmed) — consumers never see uncommitted state ([Event.ts L82–97](../../server/models/Event.ts#L82-L97)).
4. **Per-consumer job fan-out with independent retries** (confirmed) — failure isolation between notifications/webhooks/search ([worker.ts](../../server/services/worker.ts#L53-L88)).
5. **Service-modular monolith** (confirmed) — one build, six services, `SERVICES`-selected ([server/index.ts L182–190](../../server/index.ts#L182-L190)); self-hosters run one container, cloud runs specialized fleets. The deployment story *is* product design for self-hosted software.
6. **Scope-preloaded authz with loud invariants** (confirmed) — missing preloads crash with instructive messages ([document policy L283–290](../../server/policies/document.ts#L275-L308)) instead of failing open or silently denying.
7. **Watchdog tasks for user-visible async state** (confirmed) — [ErrorTimedOutFileOperationsTask](../../server/queues/tasks/ErrorTimedOutFileOperationsTask.ts) et al. sweep stuck operations into a failed state users can see.

## Risks and tradeoffs (with evidence)

1. **Event-loss window after commit** (confirmed mechanism, hypothesis on impact). Crash/Redis failure between COMMIT and `queue.add` drops the event; no relay re-reads the events table ([Event.ts L82–97](../../server/models/Event.ts#L82-L97)). Missed webhook/notification/search-index updates, silently. Likelihood low, detectability poor.
2. **Dual write paths for document content** (confirmed existence; hypothesis on edge cases). REST [documentUpdater](../../server/commands/documentUpdater.ts) vs collaborative [documentCollaborativeUpdater](../../server/commands/documentCollaborativeUpdater.ts)/[PersistenceExtension](../../server/collaboration/PersistenceExtension.ts) both mutate `content`/`state`. Convergence depends on discipline in both paths (the updater's doc comment itself warns which to use). Any new content-touching feature must pick correctly.
3. **Mega route files** (confirmed). [documents.ts](../../server/routes/api/documents/documents.ts) at 2,106 lines mixes 30 endpoints with inline query assembly. Cost: navigability, merge conflicts, temptation to copy-paste filters. (Counterpoint: locality — everything about the surface in one place; the maintainers clearly accept this.)
4. **Client cache without an invalidation policy** (confirmed shape; hypothesis on severity). Stores grow unboundedly per session; correctness relies on websocket delivery. A missed event (sleep/laptop-lid, reconnect gap) = stale UI until refetch. What's the reconnect-resync story? (*Investigate:* read the reconnection path in [WebsocketProvider](../../app/components/WebsocketProvider.tsx).)
5. **Presenter↔client-model contract is unenforced** (confirmed). No shared type between [presenters/document.ts](../../server/presenters/document.ts) output and [app/models/Document.ts](../../app/models/Document.ts) fields; drift is caught by runtime behavior/snapshots, not the compiler.
6. **`data` vs `changes` event columns mid-migration** (confirmed — the comment says so, [Event.ts L59–73](../../server/models/Event.ts#L59-L73)), with `data` being client-visible: a footgun for whoever adds event metadata next.
7. **Aging pillars** (confirmed versions): React 17, MobX 4 (decorators), react-router 5, Koa-era middleware. Not wrong — but each pins hiring docs, upgrade paths, and tool compatibility (the SWC/decorator contortions in [vitest.config.ts](../../vitest.config.ts#L20-L41) are the visible tax).

## What I'd change owning this for 3 months (prioritized)

1. **Close the event-loss window** (risk 1). Add `publishedAt` to events + a relay sweep (cron task re-enqueuing unpublished events older than N seconds). Migration: additive column; test: kill-between-commit-and-publish integration test; blast radius: small — one new task + 10 lines in `enqueue`. Highest reliability-per-effort in the repo.
2. **Contract-type the presenters** (risk 5). Derive presenter return types and export them for the client build (shared types file or codegen). Migration path: one presenter at a time, starting with document; test: `tsc` becomes the contract test. Cheap, compounding.
3. **Reconnect resync audit** (risk 4). Verify/implement "on websocket reconnect, refetch stale entities since `lastSeenAt`". Test: integration with a dropped socket. (Do the *investigate* first — this may exist.)
4. **Document the dual-write-path rules** (risk 2) as lint-able convention if possible ("content writes go through X"), plus a test asserting REST edits on actively-collaborating docs behave (define "behave" first — that definition *is* the work).
5. **Split documents.ts by concern** (risk 3) — only if the team agrees; mechanical split (info/list/search/mutations/memberships), zero behavior change, one PR per section to keep review sane. Lowest priority: style, not safety.

Each recommendation states migration path + test strategy — that habit is what separates "critique" from "complaint" in interviews and RFCs alike.

## Interview angle

- *"What would you improve in a codebase you know well?"* — deliver items 1–2 with their evidence and small blast radii. Improvements that are *shippable* impress; rewrites don't.
- *"Tell me about a tradeoff you'd defend."* — mega-route-file locality, or accepting at-least-once delivery + idempotency over exactly-once machinery.
- *"How do you evaluate an unfamiliar codebase?"* — the structure of this very page: strengths → evidenced risks → prioritized, sized changes.

## Drill

Write your own one-page critique of just the client ([app/](../../app)) in this format: 3 strengths, 3 risks with anchors, 2 prioritized changes with test strategy. Compare against risk 4/5 above. Self-grade — Strong requires: every risk has a file anchor, every change has a migration path, and at least one of your "risks" turned out on inspection to be a reasonable tradeoff you then defended instead.
