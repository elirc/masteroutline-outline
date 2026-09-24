# Runtime and Tooling Map

## Toolchain at a glance

| Concern | Tool | Where configured | Notes |
| --- | --- | --- | --- |
| Package manager | Yarn 4 (`yarn@4.11.0`) | [package.json](../../package.json#L387) | Single package — path aliases, not workspaces |
| Frontend build | Vite 8 | [vite.config.ts](../../vite.config.ts) | PWA plugin, React plugin; output consumed by the Koa server in prod |
| Server build | esbuild-style custom script | [build.js](../../build.js) invoked by `yarn build:server` | Dev backend runs from `build/` via nodemon rebuild loop ([dev:backend script](../../package.json#L14)) |
| Types | TypeScript 6, strict | [tsconfig.json](../../tsconfig.json) | Aliases: `@server/*`, `@shared/*`, `~/*` (app), `plugins/*` — mirrored for tests in [vitest.config.ts](../../vitest.config.ts#L11-L16) |
| Tests | Vitest 4, four projects | [vitest.config.ts](../../vitest.config.ts) | `server` (node + real Postgres), `app` (jsdom), `shared-node`, `shared-jsdom`. SWC transform because of legacy decorators (MobX 4, sequelize-typescript) |
| Lint/format | oxlint + oxfmt | `yarn lint`, `yarn format` ([package.json](../../package.json#L16-L19)) | Rust toolchain — fast; type-aware lint |
| Git hooks | husky + lint-staged | [lint-staged.config.mjs](../../lint-staged.config.mjs) | |
| i18n | i18next; strings extracted, never hand-added | `yarn build:i18n`, [i18next-parser.config.js](../../i18next-parser.config.js) | [shared/i18n/locales](../../shared/i18n) is generated — don't edit |
| CI | GitHub Actions | [.github/workflows/ci.yml](../../.github/workflows/ci.yml) | Plus CodeQL, Docker build, stale/auto-close bots |
| DB migrations | sequelize-cli + umzug | `yarn db:migrate`, [server/migrations](../../server/migrations) | Pending migrations block boot: `checkPendingMigrations` in [server/index.ts](../../server/index.ts#L51-L60) |
| Observability | winston logs, hot-shots (statsd), dd-trace, Sentry | [server/logging/](../../server/logging) | See [05-quality-engineering/06-observability-and-operations.md](../05-quality-engineering/06-observability-and-operations.md) |

## Runtime boundaries

Code runs in **four distinct runtimes**, and knowing which file runs where prevents a whole bug class:

1. **Browser** — everything under [app/](../../app), plus [shared/](../../shared) when imported from app. No Node APIs. Browser targets from `browserslist` ([package.json](../../package.json#L51-L53)).
2. **Node server** — [server/](../../server), plus shared. Node ≥ 20.12.
3. **Both** — [shared/](../../shared) must stay isomorphic. The editor lives here because the *server also renders/parses documents* (emails, exports, imports) using the same ProseMirror schema — see [server/editor](../../server/editor). This is the repo's most interesting runtime decision: one schema, two runtimes.
4. **Worker processes** — same Node build, different entry behavior: no HTTP handlers, just queue consumers ([server/services/worker.ts](../../server/services/worker.ts)).

The `SERVICES` env var picks which services a process runs (default all-in-one in dev: see the `dev` script listing `cron,collaboration,websockets,admin,web,worker` in [package.json](../../package.json#L13)).

## Environment configuration

Server env is a *validated class*: [server/env.ts](../../server/env.ts) (class-validator decorators; boot fails on invalid config, and `printEnv` runs at startup — [server/index.ts](../../server/index.ts#L51-L60)). Client env is the subset serialized to the browser via [server/presenters/env.ts](../../server/presenters/env.ts) and consumed by [app/env.ts](../../app/env.ts). High-level var groups (no secrets here): `DATABASE_URL`, `REDIS_URL` (+ optional `REDIS_COLLABORATION_URL`), `SECRET_KEY`/`UTILS_SECRET`, `URL`/`PORT`, S3/file storage vars, SMTP vars, per-provider OAuth credentials (consumed by plugins), `SERVICES`, `WEB_CONCURRENCY`, rate-limiter toggles.

**Transferable:** centralizing env parsing/validation (here class-validator; elsewhere Zod/envalid) is a pattern to name in interviews when asked "how do you make services fail fast?"

## Dev loops (all inferred from scripts, not executed)

```bash
make up                 # docker compose (postgres, redis) + install + SSL + dev:watch
yarn dev:watch          # backend (nodemon rebuild) + frontend (vite) concurrently
yarn test path/to.test.ts   # single test file — the loop you'll live in
make test               # recreate test DB + full suite (slow; needs docker postgres)
yarn lint && yarn format:check && yarn tsc   # what CI expects clean
yarn db:create-migration    # sequelize migration:create
```

Full command list with verification labels: [09-reference/command-cheatsheet.md](../09-reference/command-cheatsheet.md).

## Interview angle

1. *"Your app is one repo but six services — how does that work and why?"* → point at the services loop ([server/index.ts L182–190](../../server/index.ts#L182-L190)); tradeoff vs microservices (shared schema/deploy vs independent scaling of `worker`).
2. *"How do you share code between client and server safely?"* → `shared/` + isomorphism constraint + the editor-on-the-server story; failure mode: importing a Node API into shared code breaks the browser bundle.
3. *"What's your dev feedback loop and how would you speed it up?"* → nodemon rebuild vs Vite HMR asymmetry here; single-test-file discipline.
4. *"How do migrations relate to deploys?"* → boot-blocking pending-migration check; expand-migrate-contract (see [03-architecture-and-patterns/02-data-model-and-persistence.md](../03-architecture-and-patterns/02-data-model-and-persistence.md)).
