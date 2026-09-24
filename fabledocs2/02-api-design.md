# 02 — API Design: RPC Over POST, and Proud of It

## What it is and why it exists

The public API is not REST. It's RPC: `POST /api/documents.info`,
`POST /api/documents.update`, `POST /api/collections.add_user`. Every endpoint
is a POST, parameters go in the JSON body, and the route name is
`noun.verb`. Route modules live in `server/routes/api/<resource>/`, each with
a `<resource>.ts` (handlers), a `schema.ts` (Zod request schemas), and tests.

A request passes through a fixed middleware pipeline: rate limiting
(`server/middlewares/rateLimiter.ts`) → authentication
(`server/middlewares/authentication.ts`) → validation
(`server/middlewares/validate.ts`, which parses the Zod schema and populates
`ctx.input`) → an optional transaction wrapper
(`server/middlewares/transaction.ts`) → the handler. Handlers authorize via
policies, mutate via commands, and shape output via presenters
(`server/presenters/`). Every response is `{ data, policies?, pagination? }`.

Why RPC? Because the honest shape of this product's operations is verbs, not
resources. "Move this document under that one, repositioning siblings" or
"restore this revision" map terribly onto PUT/PATCH semantics. REST would have
forced us to invent sub-resources (`POST /documents/:id/moves`?) and argue
about idempotency headers. `noun.verb` is boring, greppable, and has never
once caused a design meeting.

## The hard decisions

### 1. RPC vs. REST vs. GraphQL

REST we rejected for the verb-mapping reason above. GraphQL got a real look —
the frontend's nested data needs (document → collection → memberships →
users) are exactly GraphQL's pitch. We rejected it for three reasons that
still hold: (a) our authorization is *per-object and conditional* (see
policies below), and GraphQL resolvers make it fatally easy to leak a field
through a path nobody authorized; (b) the public API is a product surface —
Zapier, integrations, curl users — and "send this exact JSON to this exact
URL" beats teaching customers a query language; (c) caching and rate limiting
per named operation is trivial with RPC.

The tradeoff that still bothers me: over-fetching. `documents.info` returns a
fat presenter payload whether you wanted one field or thirty, and the
`presentDocument` options bag in `server/presenters/document.ts`
(`includeText`, `includeData`, `includeCommentCount`...) is us slowly
reinventing field selection, one boolean at a time. That options bag is a
code smell I've made peace with.

### 2. Where authorization lives: the policy layer

Every handler does `authorize(user, "update", document)` — the implementation
is our in-house CanCan port (`server/policies/cancan.ts`) with per-model
policy files (`server/policies/document.ts`, `collection.ts`, ...). Abilities
are declared as `allow(User, "action", Model, conditionFn)` and indexed by
model + action.

The rejected alternatives: (a) role checks inline in handlers — that's how
you get the same rule written four slightly different ways; (b) row-level
security in Postgres — real isolation, but debugging RLS policies through
Sequelize-generated SQL is misery, and self-hosters would file every
permission bug against their own Postgres config; (c) a heavyweight
policy-as-data engine (OPA-style) — massive operational cost for rules that
are, at bottom, TypeScript predicates over two model instances.

The part I genuinely like: policies don't just gate requests. The *same*
ability evaluation is serialized and shipped to the client in the `policies`
key of API responses (see `app/models/Policy.ts` on the frontend). The UI
decides whether to render a "Delete" button from the same code that decides
whether DELETE succeeds. One source of truth, two consumers. The part that
bothers me: conditions can return strings (reasons) or booleans, and that
union type (`Condition` in `cancan.ts`) has confused every new engineer
including, once, me.

### 3. Validation at the door: Zod schemas with defaults

Every route declares its request shape in `schema.ts` next to the handler —
see `server/routes/api/documents/schema.ts` for the biggest example. The
`validate()` middleware parses `ctx.request` and merges into `ctx.input`, so
handlers never touch `ctx.request.body` raw. We rejected JSON Schema (the
tooling in 2020s TypeScript was clumsy and you write types twice) and
"validate in the handler" (which means you validate in *some* handlers).

The lingering annoyance: `validate.ts` throws only the *first* Zod issue as a
`ValidationError`. Fine for humans, hostile to API consumers fixing a payload
one error per request. We've never prioritized returning the full issue list
because it would change the error contract. That's tech debt with a customer-
facing interest rate.

## Evidence to go read

- `server/routes/api/documents/documents.ts` + `schema.ts` — the canonical
  resource: every pattern in this doc appears here.
- `server/middlewares/validate.ts` — 30 lines; read all of them.
- `server/policies/cancan.ts` and `server/policies/document.ts` — the engine
  and the meatiest policy file.
- `server/presenters/document.ts` — presenter options bag; note the comment on
  `includeCommentCount` about the Redis lookup cost.
- `server/middlewares/rateLimiter.ts` — note `getRateLimiterIdentifier`: it
  keys on user id when it can resolve one, IP otherwise, specifically so
  offices behind one NAT don't share a bucket.

## An incident this design prevented

A contractor once added an endpoint that fetched a document by id and returned
it — and forgot the membership check. In a handler-checks-inline world that
ships and becomes a CVE. Here it never made it past review-by-runtime: the
presenter path called into policy serialization, the policy threw
`AuthorizationError` for a non-member, and the integration test (which, per
our testing doctrine, always includes a "user from another team" case) went
red before the PR did. The layered design means forgetting authorization
produces a *loud* failure, not a silent leak. That property has paid for the
entire policy layer several times over.

## Before you move on

1. Why is every endpoint a POST, and what do we gain for rate limiting and
   auditability by naming operations `noun.verb`?
2. A response includes `policies: [...]`. Who consumes that, and what bug
   class does it eliminate compared to the frontend hardcoding role checks?
3. Add a hypothetical `documents.wordcount` endpoint: list every file you'd
   create or touch, in pipeline order, from rate limit to presenter.
