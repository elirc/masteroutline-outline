# Verification Log

Running log of what was actually inspected while building this curriculum. Anything not listed here should be treated as **inferred**, not verified.

Date: 2026-07-11. Repo state: branch `main`, clean working tree, HEAD `8e032f8cd` ("fix: Comments menu item does nothing when triggered from a document list (#12968)"). Outline version `1.9.0` per [package.json](../../package.json).

## Commands run (all read-only)

| Command | Result |
| --- | --- |
| `ls` of repo root, `server/`, `app/`, `shared/`, `plugins/`, `docs/`, `.github/workflows` | Verified monorepo layout described in [01-system-map.md](../01-codebase-cartography/01-system-map.md) |
| `ls server/routes server/routes/api server/models server/policies server/queues/{tasks,processors} server/middlewares server/presenters server/services server/collaboration server/test` | Verified backend module inventory |
| `ls app/stores app/models app/scenes app/hooks app/utils shared/editor` | Verified frontend module inventory |
| `ls server/migrations \| wc -l` | 295 migration files, from `20160619080644-initial.js` to `20260621000000-add-search-queries-user-created-index.js` |
| `grep -n '"documents\.' server/routes/api/documents/documents.ts` | Verified all ~30 document RPC endpoint names and line numbers |
| `grep -n "applicableEvents" server/queues/processors/*.ts` (selected) | Verified event subscription lists for Revisions/Notifications/SearchIndex/Backlinks processors |
| `grep -n "static findByPk\|Scopes\|withMembership" server/models/Document.ts` | Verified scope/override locations |

## Files read in full or in large part

- [package.json](../../package.json) — scripts, deps, engines, packageManager `yarn@4.11.0`
- [AGENTS.md](../../AGENTS.md) (referenced by CLAUDE.md) — project conventions
- [docs/ARCHITECTURE.md](../../docs/ARCHITECTURE.md) — official directory map
- [Makefile](../../Makefile), [vitest.config.ts](../../vitest.config.ts)
- [server/index.ts](../../server/index.ts) — process bootstrap, throng, services loop, health check
- [server/services/worker.ts](../../server/services/worker.ts) — full queue fan-out logic
- [server/services/websockets.ts](../../server/services/websockets.ts) — first ~50 lines (socket.io setup)
- [server/queues/index.ts](../../server/queues/index.ts) — queue definitions (first 80 lines)
- [server/queues/processors/BaseProcessor.ts](../../server/queues/processors/BaseProcessor.ts), [server/queues/tasks/base/BaseTask.ts](../../server/queues/tasks/base/BaseTask.ts) — full
- [server/queues/processors/RevisionsProcessor.ts](../../server/queues/processors/RevisionsProcessor.ts) — first 60 lines
- [server/queues/processors/NotificationsProcessor.ts](../../server/queues/processors/NotificationsProcessor.ts) — lines 20–60
- [server/queues/processors/SearchIndexProcessor.ts](../../server/queues/processors/SearchIndexProcessor.ts) — first 45 lines
- [server/middlewares/authentication.ts](../../server/middlewares/authentication.ts) — full
- [server/middlewares/validate.ts](../../server/middlewares/validate.ts), [server/middlewares/transaction.ts](../../server/middlewares/transaction.ts) — full
- [server/middlewares/csrf.ts](../../server/middlewares/csrf.ts) — first 40 lines
- [server/policies/cancan.ts](../../server/policies/cancan.ts), [server/policies/document.ts](../../server/policies/document.ts) — full
- [server/routes/api/documents/documents.ts](../../server/routes/api/documents/documents.ts) — lines 1–180 (imports, `documents.list`), 574–688 (`documents.info`, `documents.insights`), 1251–1360 (`documents.update`, `documents.duplicate`, `documents.move` head)
- [server/commands/documentUpdater.ts](../../server/commands/documentUpdater.ts) — full
- [server/models/Event.ts](../../server/models/Event.ts) — full
- [server/models/Document.ts](../../server/models/Document.ts) — lines 129–240 (scopes), 726–840 (`withMembershipScope`, `findByPk` override)
- [server/presenters/document.ts](../../server/presenters/document.ts) — full
- [server/test/TestServer.ts](../../server/test/TestServer.ts), [server/test/factories.ts](../../server/test/factories.ts) — first 60 lines each
- [app/utils/ApiClient.ts](../../app/utils/ApiClient.ts) — first 120 lines
- [app/stores/base/Store.ts](../../app/stores/base/Store.ts) — first 260 lines
- [app/components/WebsocketProvider.tsx](../../app/components/WebsocketProvider.tsx) — lines 110–150, 245–262

Further targeted reads/greps made while writing individual pages are noted inline in those pages when they affect a claim's confidence.

## Additional reads during the writing phase

- [server/routes/api/documents/documents.ts](../../server/routes/api/documents/documents.ts) — lines 1035–1120 (`documents.search`, both share and authenticated branches)
- [server/commands/shareLoader.ts](../../server/commands/shareLoader.ts) — first 70 lines (`loadPublicShare` validity conditions)
- [server/routes/api/comments/comments.ts](../../server/routes/api/comments/comments.ts) — `comments.create` (~lines 28–100): middleware chain, row lock, anchor-mark application; endpoint name list via grep
- [server/commands/accountProvisioner.ts](../../server/commands/accountProvisioner.ts) — first 40 lines + structural grep (team/user provisioner delegation at L132/L186, event at L210)
- [server/routes/api/index.ts](../../server/routes/api/index.ts) — first 45 lines (router mounting, shared middleware imports incl. CSRF, apiResponse)
- [server/routes/api/middlewares/pagination.ts](../../server/routes/api/middlewares/pagination.ts) — first 50 lines (`paginateQuery` count-skipping)
- [server/test/support.ts](../../server/test/support.ts) — export list (getTestServer, mockTaskSchedule, withAPIContext); [server/test/setup.ts](../../server/test/setup.ts) — first 40 lines (msw `onUnhandledRequest: "error"`, AWS SDK mocks); [server/routes/api/documents/documents.test.ts](../../server/routes/api/documents/documents.test.ts) — first 50 lines (factory/server idioms)
- Existence checks (`ls`) for every file anchored without a content read: `server/utils/fetch.ts`, `server/utils/SearchProviderManager.ts`, `app/hooks/usePolicy.ts`, `app/utils/lazyWithRetry.ts`, `server/middlewares/csp.ts`, `server/onerror.ts`, `server/routes/api/middlewares/apiErrorHandler.ts`, `server/queues/HealthMonitor.ts`, `app/models/decorators/Relation.ts` — all exist; their *contents* are cited only at the level of their role, not line numbers

## Not verified / not run

- `yarn install`, `yarn dev`, `yarn test`, `yarn build` were **not** executed (docs-only task; no dependency install performed). All command documentation in [command-cheatsheet.md](command-cheatsheet.md) is marked verified-by-reading-scripts, not verified-by-execution.
- Database migrations were not run; schema claims come from model definitions and migration filenames.
- No plugin code was read in depth (`plugins/*`) except directory listings; claims about specific plugins are labeled inferred.
- Runtime behavior of the collaboration (Hocuspocus) service is described from code reading only.

## 2026-10-06 accuracy pass

Static re-check against `elirc/masteroutline-outline` (branch `training`, commit `05f615f`); nothing was installed or run.

- All relative Markdown links in `fabledocs*/`, `learn/` and `training/` resolve, and every `#Lx-Ly` anchor is within its file. One link in `05-quality-engineering/04-performance-thinking.md` pointed at `DocumentHelper.ts`; the file is `server/models/helpers/DocumentHelper.tsx`.
- `learn/` linked an apprenticeship curriculum outside this repository and a `training/navigation/04-upstream-pr-dry-run.md` that was never committed; those references are now marked as external or missing.
- Spot-read and still accurate: `documents.info` at `server/routes/api/documents/documents.ts:573`, `Event.schedule` at `server/models/Event.ts:140`, `dev:watch` at `package.json:15`, the CI `postgres:14.2` service, and the `ioredis` mock in `server/test/setupMocks.ts:37`.

## Known uncertainties

- Exact behavior of `documents.update.debounced` event scheduling (DebounceProcessor was not read in full) — flagged where mentioned.
- The interaction between the collaborative editing path (Yjs state) and the REST `documents.update` path is described from `documentUpdater`/`documentCollaborativeUpdater` file naming and partial reads; edge cases labeled "investigate".
- Line numbers were correct at HEAD `8e032f8cd`. They will drift as the repo changes; anchors include enough context (function names) to relocate.
