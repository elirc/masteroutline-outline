# Frontend Framework Question Cards

Twelve cards on React + MobX + SPA architecture, anchored to this repo's UI. Format per [module README](README.md): answer aloud in 90s citing the anchor.

## Q1: "How does your state management decide what re-renders?"
Round: frontend. Testing: do you understand your reactivity model or just use it.
Repo anchor: `observer` components over observable stores ([Store.ts data maps](../../app/stores/base/Store.ts#L56-L60)).
Junior: "MobX tracks changes." Mid: tracking happens per *dereference during render* — access `doc.title` in render and only title changes re-render this component; contrast Redux (selector + snapshot diff). Senior: identity preservation as the enabling invariant — `add` merges into existing instances ([L152–172](../../app/stores/base/Store.ts#L152-L172)) so subscriptions survive updates; the failure mode (destructure-too-early) and how you'd detect it.
Follow-ups: "why not React Query?" → server-push model favors normalized entity cache ([Pattern 13](../03-architecture-and-patterns/05-pattern-catalog.md)).

## Q2: "Where should state live?"
Round: frontend. Repo anchor: stores for domain state ([app/stores](../../app/stores)), [UiStore](../../app/stores/UiStore.ts) for UI chrome, component `useState` for ephemeral local.
Junior: "lift state up." Mid: the three-tier rule with examples: document title (domain → store), sidebar collapsed (app UI → UiStore), input draft before submit (local). Senior: the *cost gradient* — store state is observable everywhere but needs lifecycle discipline; local state is free but unshareable; put state at the narrowest scope that serves every reader.

## Q3: "Explain stale closures in hooks and how you avoid them."
Round: frontend. Repo anchor: conceptual; MobX reduces the problem (observables are read live, not captured) — a real architectural answer.
Junior: recites the setInterval example. Mid: deps arrays as capture lists; functional updates; refs for latest-value escape hatches. Senior: this codebase's structural avoidance — components read observables at render time and effects are rarer because server push updates stores directly ([WebsocketProvider](../../app/components/WebsocketProvider.tsx#L253-L262)); when effects do capture, `reaction`/`autorun` are the MobX-native fix.

## Q4: "How do you handle data fetching, loading, and caching?"
Round: frontend. Repo anchor: store fetch methods converging on `add` ([DocumentsStore](../../app/stores/DocumentsStore.ts)); `isFetching/isLoaded` flags ([Store.ts L60–67](../../app/stores/base/Store.ts#L60-L68)); request dedupe in [ApiClient L62–64](../../app/utils/ApiClient.ts#L62-L64); pagination via [usePaginatedRequest](../../app/hooks/usePaginatedRequest.ts).
Junior: useEffect + fetch + setState. Mid: centralized cache with dedupe, single write path, error mapping to typed errors. Senior: staleness policy as the honest gap (no TTL; freshness depends on websockets — [critique risk 4](../03-architecture-and-patterns/06-architecture-critique.md)) and what reconnect-resync would add.

## Q5: "Optimistic updates — when and how?"
Round: frontend. Repo anchor: [Kata 7](../04-code-reading-gym/04-review-katas.md) covers the design; check a real store's save flow for the current stance.
Junior: "update UI first, rollback on error." Mid: rollback must restore *exactly* (cascades, paranoid deletes); echo events must be idempotent. Senior: choose per action by (reversibility × latency × failure rate); destructive actions pessimistic, toggles optimistic; websocket echo makes "rollback" racy — design for convergence, not undo.

## Q6: "How does the UI know what the user is allowed to do?"
Round: frontend. Repo anchor: [usePolicy](../../app/hooks/usePolicy.ts) + server-serialized abilities ([Pattern 14](../03-architecture-and-patterns/05-pattern-catalog.md)).
Junior: "check the user's role." Mid: server ships per-entity ability maps; UI renders them; server re-enforces on action — no duplicated rules. Senior: staleness handling (mutations re-ship policies), and the security framing: client checks are UX; the 403 path must still be graceful.

## Q7: "Routing and code-splitting strategy?"
Round: frontend. Repo anchor: lazy route chunks ([app/routes](../../app/routes/index.tsx)), `lazyWithRetry` handling stale-chunk-after-deploy ([app/utils/lazyWithRetry.ts](../../app/utils/lazyWithRetry.ts)).
Junior: React.lazy + Suspense. Mid: split by route; the deploy failure mode (hashed chunk vanished) and the retry/reload mitigation. Senior: bundle budgets in CI (relative-ci) making regressions reviewable; heavyweights (mermaid/katex) as the next split targets.

## Q8: "How do you keep a long-lived SPA session correct?"
Round: frontend. Repo anchor: websocket entity events ([WebsocketProvider](../../app/components/WebsocketProvider.tsx)), auth expiry via ApiClient's unauthorized handler ([L82–91](../../app/utils/ApiClient.ts#L82-L91)), `UpdateRequiredError` forced-upgrade path.
Junior: hadn't considered it. Mid: push updates + session-expiry handling + version headers. Senior: the full skew story — old tabs across deploys receive new payloads; reconnect gaps need resync; memory growth in stores; this is the question where naming *three* concrete mechanisms separates you.

## Q9: "Forms and validation client-side vs server-side?"
Round: frontend. Repo anchor: react-hook-form is present ([package.json](../../package.json)); server Zod is authoritative ([validate.ts](../../server/middlewares/validate.ts)).
Junior: validate in the form. Mid: client validation = UX, server = truth; mirror the *rules* not the code; render server 400s into field errors (the error message format `field: message` from validate.ts is parseable — is that used? *investigate*). Senior: shared schema aspirations (Zod on both sides) and why this repo doesn't (schemas live server-side; shared/validations.ts holds some — check before claiming).

## Q10: "Accessibility — what do you actually do?"
Round: frontend. Repo anchor: Radix primitives throughout ([package.json — @radix-ui/*](../../package.json#L91-L104)), roving tabindex fork (`@getoutline/react-roving-tabindex`), AGENTS.md a11y mandate ([L96](../../AGENTS.md#L88-L97)).
Junior: "alt tags and aria-labels." Mid: use accessible primitives (Radix gives focus management/keyboard/ARIA for free); keyboard-first testing; semantic HTML before ARIA. Senior: a11y as architecture — an editor + command palette (kbar) app lives or dies by focus management; the fact that they maintain a fork of a roving-tabindex lib tells you where the pain was.

## Q11: "Styling at scale — tradeoffs of your approach?"
Round: frontend. Repo anchor: styled-components + theme ([shared/styles](../../shared/styles), [useBuildTheme](../../app/hooks/useBuildTheme.ts)).
Junior: "CSS-in-JS is convenient." Mid: co-location + theming + dead-style elimination vs runtime cost and SSR complexity; contrast Tailwind/CSS-modules honestly. Senior: the migration-cost lens — styling choices are 5-year commitments (this repo shows the age: styled-components 5, React 17); evaluate on exit cost, not entry ergonomics.

## Q12: "How would you debug a UI that shows stale data?"
Round: frontend (bridges to debugging round). Repo anchor: the full method is [debugging Scenario 1](../05-quality-engineering/03-systematic-debugging.md).
Junior: "check the network tab." Mid: the transport-vs-merge binary search: WS frames → room membership → store merge → observer wrapping. Senior: names the class (cache-consistency across push/pull) and the instrumentation you'd leave behind (Sentry breadcrumbs on WS events already exist — [WebsocketProvider L115–122](../../app/components/WebsocketProvider.tsx#L110-L125)).

---

Grading: as always — Basic defines, Solid cites this repo, Strong adds the failure mode + alternative. Cards Q1, Q6, Q8 are the highest-yield for mid-level loops; over-practice those.
