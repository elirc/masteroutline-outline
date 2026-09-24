# Command Cheatsheet

All commands read from [package.json](../../package.json#L6-L39), [Makefile](../../Makefile), and [AGENTS.md](../../AGENTS.md) — **inferred** (verified by reading the scripts, not by execution) unless noted. Requires Node 20/22, Yarn 4, Docker.

## Setup and dev

| Command | Does | Notes |
| --- | --- | --- |
| `make up` | postgres+redis containers, install, local SSL, dev watch | the one-command start |
| `docker compose up -d redis postgres` | just the datastores | |
| `yarn install --immutable` | deps (Yarn 4, lockfile-strict) | `postinstall` runs patch-package |
| `yarn dev:watch` | backend (nodemon rebuild loop) + frontend (Vite) | backend serves API; Vite serves SPA with HMR |
| `yarn dev:backend` / `yarn vite:dev` | each half separately | |
| `node --inspect` debugging | already wired into the `dev` script | attach Chrome/VS Code debugger |

Services selection: the dev script runs `--services=cron,collaboration,websockets,admin,web,worker`; trim to isolate behavior (e.g. no `worker` = watch async features stop).

## Build / run production-style

| Command | Does |
| --- | --- |
| `yarn build` | clean → vite build → i18n extraction → server build |
| `yarn start` | run `build/server/index.js` |
| `docker compose build outline` (`make build`) | production image |

## Quality gates (what CI expects clean)

| Command | Does |
| --- | --- |
| `yarn lint` | oxlint, type-aware, over app/server/shared/plugins |
| `yarn format` / `yarn format:check` | oxfmt write / verify |
| `yarn tsc` | typecheck (per AGENTS.md; runs the TS compiler) |
| `yarn lint:changed` | lint only files changed vs git |

## Tests

| Command | Does | Notes |
| --- | --- | --- |
| `make test` | drop/create/migrate **test DB**, run everything | one-time DB setup lives here — run before first `yarn test` |
| `yarn test path/to/file.test.ts` | single file | the loop you'll live in (per AGENTS.md, prefer this) |
| `yarn test:server` / `:app` / `:shared` | per vitest project | slow; avoid habitually |
| `yarn test:watch` (`make watch`) | watch mode | |

Server tests hit a real Postgres (`NODE_ENV=test` DB) with S3/network mocked ([testing strategy](../05-quality-engineering/01-testing-strategy.md)).

## Database

| Command | Does |
| --- | --- |
| `yarn db:create-migration --name=add-x-to-y` | scaffold a migration (sequelize-cli) |
| `yarn db:migrate` | apply pending (also required at boot — server refuses to start with pending migrations) |
| `yarn db:rollback` | undo last |
| `yarn db:reset` | drop/create/migrate (destructive — dev only) |
| `NODE_ENV=test yarn sequelize db:migrate` | test-DB variant (see Makefile) |

## i18n and misc

| Command | Does |
| --- | --- |
| `yarn build:i18n` | extract translation strings (never hand-edit locales) |
| `yarn install-local-ssl` | dev HTTPS certs |
| `yarn upgrade` | pull + install + build + migrate (self-hoster update path) |

## Everyday git archaeology (verified — used while building these docs)

```bash
rg --files | head              # inventory
rg -n "applicableEvents" server/queues/processors   # who reacts to an event
git log --oneline -20          # recent work
git show <sha>                 # review a merged change (see 07-career/01 drill)
git log -S "someSymbol" --oneline   # when did this symbol appear/change
```
