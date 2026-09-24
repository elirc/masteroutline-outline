# ADR-004: Centralized CanCan Policy Layer, Serialized to Clients

**Status:** Active

## Context

Authorization here is per-object and conditional, not role-lookup: whether a
user can update a document depends on team membership, collection permission,
direct document membership, group membership, share settings, and archival
state — composed. The same rules must gate API requests **and** drive UI
affordances (does this user see a "Delete" menu item?). Two implementations
of the same rules is how frontends lie about what the backend will allow.

## Decision

A single in-house authorization engine adapted from the `cancan` package:
`server/policies/cancan.ts` provides `allow(Model, actions, targets,
condition)`; per-model policy files (`server/policies/document.ts`,
`collection.ts`, ...) declare abilities as TypeScript predicates over
(performer, target). Handlers call `authorize(user, "update", document)`,
which throws `AuthorizationError` on failure — authorization is an assertion,
not a query. Abilities are indexed by model + action for lookup.

Crucially, the same ability evaluation is **serialized into API responses**
(the `policies` key) and hydrated into the frontend's `app/models/Policy.ts`;
UI components consult the shipped policy object rather than re-deriving
permissions from roles.

## Alternatives Considered

1. **Inline role checks in handlers.** Rejected: the same rule ends up
   written N slightly-different ways; a forgotten check fails silent-open.
2. **Postgres row-level security.** Rejected: real defense-in-depth, but
   debugging RLS through ORM-generated SQL is brutal, and self-hosters would
   misattribute permission bugs to their own database configuration.
3. **External policy engine (OPA-style) or policy-as-data.** Rejected: heavy
   operational surface for rules that are ultimately small TypeScript
   predicates over two loaded model instances; would also break the "ship
   policies to the client" trick, which depends on evaluating abilities
   in-process against hydrated models.

## Consequences

**Positive:**
- One source of truth, two consumers: the code that gates DELETE also decides
  whether the button renders. Frontend/backend permission drift is
  structurally impossible for covered actions.
- Forgetting authorization tends to fail **loud** (a policy-driven presenter
  path throws) rather than leak — reinforced by the testing convention that
  every authz test includes a cross-team actor (ADR-010).
- Policy files are greppable, reviewable TypeScript — no DSL to learn.

**Negative:**
- Conditions may return `boolean | string` (string = denial reason) — the
  union type in `cancan.ts` confuses every newcomer.
- Policy evaluation happens on hydrated models, so authorization cannot be
  pushed into queries; list endpoints must be careful that their `WHERE`
  clauses match what policies would permit (two places to keep aligned —
  the real residual risk of rejecting RLS).
- Serialized policies bloat every response for clients that don't need them.

## Evidence

- `server/policies/cancan.ts` — the engine; note the ability index and the
  `Condition` union type.
- `server/policies/document.ts` — the most complex composed policy.
- `server/policies/collection.test.ts`, `document.test.ts` — policies are
  unit-tested directly, unusual and telling.
- `app/models/Policy.ts` and `abilities` usage in `app/components`/menus —
  the client-side consumer.
- `server/services/websockets.ts` importing `can` from `@server/policies` —
  the same engine gating websocket room joins, not just HTTP.
