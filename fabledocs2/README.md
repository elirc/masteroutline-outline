# Onboarding: From the Engineer Who Built This Thing

Welcome. You're joining a codebase that is roughly ten years old in spirit — a
TypeScript monolith that serves a real-time collaborative wiki to teams, ships
as a single Docker image, and has to work equally well on our cloud and on some
ops person's Raspberry Pi. That dual life (SaaS + self-hosted) explains about
half of the weird decisions you're going to ask me about, so keep it in mind
as a lens for everything below.

This pack is written the way I'd explain the system over a week of coffee
chats: what each subsystem does, the decisions that were genuinely hard, the
alternatives we walked away from, and the scars. It is not marketing copy.
Where the design still bothers me, I say so.

## How to read this

In order. Each doc builds on the previous one, and each ends with three
questions. Don't skip the questions — if you can't answer them, re-read before
moving on, because the next doc assumes you can.

| Doc | Subsystem | The one-sentence version |
| --- | --- | --- |
| [01-architecture.md](01-architecture.md) | Architecture | One process image, six services behind a flag, plugins for everything optional, and an events table as the spine. |
| [02-api-design.md](02-api-design.md) | API design | RPC-over-POST, not REST — on purpose — with Zod at the door, policies in the middle, and presenters at the exit. |
| [03-data-access.md](03-data-access.md) | Data access | Sequelize with decorators, a transaction-per-mutation middleware, and a document that exists in three representations at once. |
| [04-testing.md](04-testing.md) | Testing strategy | Vitest projects per runtime, factories over fixtures, and real HTTP through TestServer instead of mocked routers. |
| [05-error-handling.md](05-error-handling.md) | Error handling | Typed error constructors with an `isReportable` bit, one Koa onerror funnel, and graceful shutdown as a first-class concern. |
| [06-observability.md](06-observability.md) | Observability | Category-tagged Winston logs, StatsD counters, dd-trace spans via decorator, and Sentry only for what we chose to report. |

## Ground rules that will save you pain

1. **`shared/` runs in both the browser and Node.** Never import anything
   server-only into it. The editor schema lives there precisely because both
   sides must agree on it (`shared/editor/`).
2. **Mutations flow through commands and emit Events.** If you write to the
   database from a route handler directly, you've probably skipped a step that
   websockets, webhooks, notifications, and search indexing all depend on.
3. **Every API mutation runs in a transaction** (`server/middlewares/transaction.ts`).
   Pass `ctx.state.transaction` to every DB call inside the request, or you'll
   create the classic "half the write committed" bug.
4. **Authorization is `authorize(user, "action", model)` or it didn't happen.**
   The policy layer (`server/policies/`) is the only authz source of truth.
5. **The Yjs state blob is the source of truth for document content.** The
   markdown `text` and JSON `content` columns are projections. Treat them as
   caches that can be stale.

## What I'd tell you on day one

Run it locally before reading any more docs: `yarn install`, copy `.env.sample`,
`yarn db:migrate`, `yarn dev:watch`. Open two browsers, edit the same document
in both, and watch the cursors. Almost every hard problem in this codebase —
persistence, conflict resolution, presence, offline sync — exists in service
of that one demo. When a design choice seems baroque, ask yourself "what does
this do for two people typing in the same paragraph?" and it usually makes
sense.
