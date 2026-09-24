# ADR-001: Modular Monolith with Runtime Service Composition

**Status:** Active-with-Scar-Tissue

## Context

The product ships to two audiences with opposite operational profiles:
self-hosters who want `docker run` + Postgres + Redis and nothing else, and a
cloud deployment that needs to scale the API, websockets, background workers,
and the real-time collaboration server independently. The collaboration
workload (long-lived connections, in-memory Y.Doc state, CPU spikes on large
documents) is especially unlike the request/response API workload.

## Decision

Build one deployable artifact containing every service, and compose at
runtime: `server/index.ts` parses `--services=cron,collaboration,websockets,admin,web,worker`
(or the `SERVICES` env var) and starts only the requested subset. The service
registry is `server/services/index.ts`; each service (`web.ts`,
`websockets.ts`, `collaboration.ts`, `worker.ts`, `cron.ts`, `admin.ts`) is a
function that attaches to a shared Koa app and HTTP server. Per-container
process fan-out is handled by `throng` in `server/index.ts`.

## Alternatives Considered

1. **True microservices** (separate deployables, at minimum splitting
   collaboration out). Rejected: the collaboration server needs the exact
   same Prosemirror schema (`shared/editor/`), models, and policy checks as
   the API; separate deployables make schema version-skew a first-class
   failure mode, and self-hosters would need to orchestrate N containers.
2. **Pure single-process monolith** (no service flags). Rejected: cloud
   cannot scale websocket fan-out and CPU-bound collaboration independently
   of API throughput, and a worker stall would degrade interactive traffic.
3. **Serverless/functions for background work.** Rejected implicitly: nothing
   in the repo assumes a FaaS runtime, and self-hosting rules it out as the
   common path.

## Consequences

**Positive:**
- One build artifact (`Dockerfile`, `yarn build` → `build/server/index.js`)
  serves both audiences; deployment topology is configuration, not code.
- All services share models, policies, and the editor schema by import —
  zero cross-service API surface to version.
- Self-host default (`yarn dev` runs all services in one process) is trivial.

**Negative:**
- No fault isolation on single-container installs: a collaboration memory
  leak takes the API down with it.
- Shared-fate dependencies: every service pays the boot cost of all code.
- The scar: running `collaboration` without `REDIS_COLLABORATION_URL` forces
  `webProcessCount = 1` (top of `server/index.ts`) because Y.Doc state is
  process-local — a runtime clamp guarding a data-integrity invariant. The
  system degrades quietly instead of refusing loudly; self-hosters rarely
  discover why they're single-process.

## Evidence

- `server/index.ts` — `getArg("services")`, `throng`, the
  `webProcessCount = 1` clamp with its explanatory log line.
- `server/services/index.ts` and the six service modules.
- `package.json` `dev` script — the full service list in one process.
- `Procfile` / `docker-compose.yml` — the two deployment shapes from one image.
