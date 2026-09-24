# Type System and Contracts

TypeScript's job in this repo: make **contracts** (agreed shapes between parts) machine-checked wherever possible, and make the unchecked seams *visible*. Knowing where each seam is = knowing where bugs can still live.

## Where types are strong here

- **Request contracts:** Zod schemas ([documents/schema.ts](../../server/routes/api/documents/schema.ts)) both validate at runtime and generate the handler's input type (`APIContext<T.DocumentsUpdateReq>` — [documents.ts L1255](../../server/routes/api/documents/documents.ts#L1251-L1258)). One definition, two enforcement layers — the gold standard. Transferable: any boundary type that isn't backed by a runtime check is a *hope*, not a contract.
- **Assertion functions:** `authorize(): asserts target` ([cancan.ts L182–191](../../server/policies/cancan.ts#L182-L191)) teaches the compiler "past this line, `document` is non-null" — control flow encoded in types. Same family: type guards, `is` predicates.
- **Shared enums as wire vocabulary:** `UserRole`, `DocumentPermission`, `StatusFilter` in [shared/types.ts](../../shared/types.ts) used by schema validation, policies, and UI — a three-way contract from one definition.
- **Generic infrastructure:** `Store<T extends Model>` ([Store.ts L56](../../app/stores/base/Store.ts#L53-L96)) and `BaseTask<T extends object>` ([BaseTask.ts L11](../../server/queues/tasks/base/BaseTask.ts#L11-L27)) — generics for *infrastructure reuse*, the highest-value generics use. Note `PartialExcept<T, "id">` ([app/types.ts](../../app/types.ts)) — a custom mapped type expressing "partial updates must at least identify themselves." Reading utility types like this fluently is a mid-level marker.
- **Overloads encoding behavior:** `findByPk`'s dual signature — `NonNullFindOptions` returns `Document`, plain options return `Document | null` ([Document.ts L758–765](../../server/models/Document.ts#L750-L770)) — `rejectOnEmpty` changing the return type is subtle, good API design.

## Where types are weak (the honest map)

- **ORM boundary:** scopes and includes are stringly composed ([withMembershipScope](../../server/models/Document.ts#L734-L748)); whether `memberships` is loaded is invisible to types → enforced by runtime `invariant` ([document policy L283–290](../../server/policies/document.ts#L275-L308)). Lesson: when types can't express an invariant, *crash loudly* rather than hoping.
- **Presenter → client model seam:** presenter returns `Record<string, unknown>` ([document.ts L57](../../server/presenters/document.ts#L28-L60)); nothing type-checks it against [app/models/Document.ts](../../app/models/Document.ts). Drift is possible; see [critique](../03-architecture-and-patterns/06-architecture-critique.md) recommendation 2.
- **Queue payloads:** job data is JSON through Redis; `as Event` casts on the consumer side ([worker.ts L30](../../server/services/worker.ts#L29-L41)). Cross-process contracts can't be compiler-checked — version skew between enqueuer and consumer is a real (rare) failure mode.
- **Marked debts:** `@ts-expect-error TODO` ([cancan.ts L248](../../server/policies/cancan.ts#L243-L254)), variance-forced `any` with an explanatory comment ([cancan.ts L5–6](../../server/policies/cancan.ts#L1-L10)). House style per [AGENTS.md](../../AGENTS.md#L61-L69): no `any`, avoid assertions — so every suppression carries a justification. A suppression *with a reason* is engineering; without one it's decay.

## Idioms to practice (with repo grounding)

1. **Narrowing by predicate:** the auth middleware's `OAuthAuthentication.match(token)` branches ([authentication.ts L156–247](../../server/middlewares/authentication.ts#L156-L247)) — runtime matcher per branch, each branch fully typed.
2. **`unknown` at edges, never `any`:** caught errors are `unknown` → normalized via `toError` helpers (grep `toError` — used in [server/index.ts](../../server/index.ts#L1-L5), worker, middleware).
3. **Discriminated unions for events:** `Event` types in [server/types.ts](../../server/types.ts) keyed by `name` — processors switch on it ([NotificationsProcessor](../../server/queues/processors/NotificationsProcessor.ts#L42-L60)).
4. **`satisfies`-era thinking:** this repo predates heavy `satisfies` use; know it anyway for interviews — "validate a literal against a type without widening."

## Interview angle

Cards in [08-interview-prep/01](../08-interview-prep/01-js-ts-node-deep-dive.md) (Q5, Q6, Q13):
1. "How do you keep TS types honest at runtime boundaries?" → Zod-inferred types; hope-vs-contract line.
2. "Explain an assertion function you've used." → `authorize` narrowing.
3. "`unknown` vs `any`; when are assertions OK?" → edges + justified suppressions.
4. "How do you type a generic store/repository?" → `Store<T>`, `PartialExcept`.
5. "Where can't TypeScript save you?" → ORM includes, cross-process payloads, presenter drift — naming *specific* unfixable seams beats generic praise of strict mode.

## Drill

Write (in a scratch file) the type signature you'd give a `presentDocument` that *guarantees* its output matches a `DocumentResponse` shared with the client, then list what breaks in the current code if you enforced it. This is literally recommendation 2 of the [critique](../03-architecture-and-patterns/06-architecture-critique.md) — doing the type-level design is senior kata material.
