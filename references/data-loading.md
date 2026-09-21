# Data Loading in Code Apps

Use this reference when a page, route, tab, modal, provider, hook, service, or repository loads multiple resources or feels slow.

## Core principle

Make the first useful view available as soon as its required data is ready. Optional data may load lazily or be prefetched, but it must not extend the critical loading boundary.

**Decision gate:** before editing, produce a request plan with one row per resource: initial visibility, whether initial rendering or authorization requires it, activation signal, connector cost, reuse/deduplication, and loading-state owner. Unknown evidence produces an interaction-bound default. Present any proposed eager loading of hidden resources as a tradeoff requiring user confirmation. Time pressure and the absence of a caching dependency do not remove this gate.

Parallel is not the same as background. Requests remain blocking when one `Promise.all`, Suspense boundary, or `loading` flag waits for every result.

## Classify each request

Trace each request from UI through hook, service, repository, and generated connector.

| Class | Example | Default |
|---|---|---|
| Critical | Data and authorization required for the initial useful view | Load immediately; may control the page loader |
| Visible section | Data for an independently usable section already on screen | Load independently with a local state |
| Interaction-bound | Hidden tab, modal, accordion, or optional action | Load on intent or activation |
| Likely next | Small, reusable data with a strong probability of use | Prefetch after the critical view is usable |
| Speculative | Expensive or rarely used data | Do not preload |

Do not interpret "load only what is needed now" as a ban on prefetching. Prefetch only when the initial view is already usable, the next interaction is probable, connector cost is acceptable, and the result is cached or otherwise reused.

A request to make every later interaction instant is not evidence that every hidden resource is likely next. Do not remove lazy gates solely to satisfy that proposal. When probability or connector cost is unknown, keep the resource interaction-bound and state what measurement could justify prefetching it.

## Loading boundaries

Independent resources need independent data, loading, error, empty, retry, and stale-response handling.

```tsx
const detail = useDetail(id); // owns the page loading boundary
const contacts = useContacts(id, activeTab === 'contacts');
const candidates = useCandidates({ enabled: ownerModalOpen });
```

Do not combine those resources behind one page-wide spinner. A local failure must not replace unrelated usable content. If authorization depends on deferred data, keep the affected action unavailable until that data resolves; never grant access from an optimistic fallback.

Prefer intent signals such as hover, focus, modal intent, or known navigation over unconditional mount-time requests. `setTimeout(..., 0)` defers JavaScript execution but does not create low-priority networking.

Deduplicate in-flight work and reuse prefetched results. A later duplicate request is not successful prefetching.

## Trace hidden connector cost

A single service method may hide a waterfall:

```text
detail -> role assignments -> one user request per assignment
```

Inspect the complete request graph. Replace sequential independent work with parallel work, select only required columns, filter at the connector, and use bulk reads or explicit read models where supported. Avoid calling a generated connector once per record.

Keep repositories focused on one resource or deliberate read model. Compose cross-resource results in a service or explicit use case so their cost and failure behavior remain visible.

Avoid broad route providers that load unrelated resources for every child page. Split provider ownership when children have different readiness requirements; do not fix incomplete provider state by making every child wait for every resource.

## Verification

Use representative Code Apps or Dataverse latency and inspect the actual request graph. Verify that:

- the initial view becomes usable without hidden or optional data;
- a tab or modal triggers at most one reusable request;
- local failures preserve unrelated content;
- no repeated per-record connector pattern remains;
- prefetch does not delay critical requests;
- performance claims include comparable before/after timings, or are labeled as static hypotheses when runtime measurement is unavailable.

## Red flags

- `Promise.all` mixes critical and optional requests;
- one loading boolean represents independently usable sections;
- every hidden tab fetches on mount without evidence;
- a provider loads all resources for all child routes;
- a repository constructs another resource repository;
- a loop performs connector reads one record at a time.

## Common mistakes

| Reasoning | Correction |
|---|---|
| "They run in parallel, so they cannot delay the page" | They delay any boundary that awaits all results. |
| "Preload every tab so later clicks are instant" | Prefetch only likely-next data after the critical view is usable. |
| "Wait for every provider resource to avoid incomplete state" | Give each consumer an explicit resource readiness contract. |
| "The service is one call, so there is no N+1" | Trace through repositories to the generated connector calls. |
