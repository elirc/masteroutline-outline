# Validation, Authentication, and Permissions

Three different questions, three different layers — collapsing them is the junior mistake this page exists to prevent:

- **Validation:** is the request well-formed? (Shape, types, ranges.)
- **Authentication (authn):** who is calling? (Identity.)
- **Authorization (authz):** may *this* identity do *this* to *this object*? (Permission.)

## The gauntlet a mutating request runs

Using [documents.update](../../server/routes/api/documents/documents.ts#L1251-L1294) as the reference:

| # | Gate | Where | Rejects with |
| --- | --- | --- | --- |
| 0 | Global rate limit | [defaultRateLimiter at app level](../../server/index.ts#L97-L99) | 429 |
| 1 | CSRF verification (cookie-auth mutations) | [csrf.ts](../../server/middlewares/csrf.ts) via [api/index.ts](../../server/routes/api/index.ts) | 403 CSRFError |
| 2 | Authentication (3 token types, transport rules) | [authentication.ts](../../server/middlewares/authentication.ts#L135-L247) | 401 |
| 3 | Account state (suspension) + role floor | [authentication.ts L249–260](../../server/middlewares/authentication.ts#L249-L260) | 403 |
| 4 | Per-route rate limit (where present) | e.g. [comments.create 25/min](../../server/routes/api/comments/comments.ts#L28) | 429 |
| 5 | Schema validation | [validate.ts](../../server/middlewares/validate.ts) + [documents/schema.ts](../../server/routes/api/documents/schema.ts) | 400 |
| 6 | Resource-scoped load | `Document.findByPk(id, { userId })` ([override](../../server/models/Document.ts#L758-L833)) | 404 via rejectOnEmpty / null |
| 7 | Authorization | `authorize(user, "update", document)` → [policy](../../server/policies/document.ts#L86-L102) | 403 |
| 8 | Semantic validation | in-handler/command checks (e.g. "cannot duplicate into a draft", [L1331–1333](../../server/routes/api/documents/documents.ts#L1324-L1334)) | 400 |

Notice the ordering logic: cheap and identity-free gates first; object-specific gates after loading. And note gate 6+7 together answer IDOR (below).

## Authorization model in depth

Grant sources that policies combine (all preloaded by scopes — [Pattern 5](05-pattern-catalog.md)):
1. Team membership + role (admin/member/viewer/guest — role floors also enforceable at the middleware level via `auth({ role })`).
2. Collection access: default collection permission or explicit `UserMembership`/`GroupMembership` on the collection.
3. Document-level memberships: user or group, with `DocumentPermission` levels (Read / ReadWrite / Admin) — see `includesMembership` ([document policy L275–308](../../server/policies/document.ts#L275-L308)).
4. Special cases: draft author rights ([read policy L30](../../server/policies/document.ts#L21-L34)), team preferences gating features (commenting — [L53–64](../../server/policies/document.ts#L53-L64)), team mutability (`isTeamMutable` — suspended/demo teams).

The composition style is readable boolean algebra via `and`/`or` helpers ([policies/utils.ts](../../server/policies/utils.ts)) — policies read like specifications. `update` ⊃ `read` (must read to update), `publish` ⊃ `update` + draft, `archive` adds admin-ish requirements. Learning to *read* this lattice quickly transfers to every cancan/pundit/casl codebase.

## IDOR, and why this repo mostly doesn't have it

An **IDOR** (insecure direct object reference) is "I changed the id in the request and got someone else's object." Defense here is structural, two layers deep:
- Loads are *scoped to the caller* (`{ userId }` loads memberships; team filters in list queries), so foreign objects often come back with no grant data;
- `authorize` then requires an actual grant, and `isTeamModel` checks the tenant match ([policies/utils.ts](../../server/policies/utils.ts)).

The residual risk sites a reviewer checks: endpoints taking *related* ids (e.g. `collectionId`, `parentDocumentId` on create/move/duplicate) — each must be separately authorized. [documents.duplicate](../../server/routes/api/documents/documents.ts#L1296-L1352) does exactly this: three loads, three `authorize` calls. **That triple-check is the house pattern; a new endpoint missing one is the highest-value review catch in this codebase.**

## What a junior misses vs what a senior checks

| Junior misses | Senior checks |
| --- | --- |
| Client-side `can.update` is UX, not security | Every mutation has a server `authorize` for **each** object id in the payload |
| 404 vs 403 semantics | Scoped loads return "not found" for foreign tenants — no existence oracle |
| Validation ≠ authorization | Zod can't know your permissions; policy can't fix malformed input |
| Optional auth endpoints | Every line after `auth({optional:true})` handles `user == null` ([documents.info](../../server/routes/api/documents/documents.ts#L574-L651)) |
| Public shares are "just a flag" | Share flows re-derive publicness (`cannot(user,"read")`), strip fields, fence subtrees ([Flow 2](../01-codebase-cartography/05-key-flows.md)) |
| Events/webhooks carry data too | `Event.data` is client-visible by design ([Event.ts L59–65](../../server/models/Event.ts#L59-L65)) — don't stuff secrets into events |

## Interview angle

1. *"Explain authn vs authz with a real example"* — the gauntlet table, gates 2 vs 7.
2. *"How do you prevent IDOR in a multi-tenant app?"* — scoped loads + per-object authorize + the duplicate-endpoint triple-check story.
3. *"Where should validation live?"* — layered: schema (shape) → semantic checks (state-dependent) → DB constraints (last line). Point at gates 5 and 8 being different lines of code.
4. *"How do viewer/guest roles degrade functionality?"* — policy conditions like `!actor.isGuest` and team preference gates; UI follows automatically via serialized policies.

## Drill

Threat-model `documents.move` ([L1354+](../../server/routes/api/documents/documents.ts#L1354-L1400)): list every id the request carries, and for each, which gate stops a hostile value. Then verify against the code. Self-grade — Basic: found the ids. Solid: matched each to a specific authorize/scope. Strong: constructed one hostile combination the gates *don't* obviously cover and designed the test proving it's safe (or filed it as an investigate item).
