# PostHog contributor onboarding

This guide is optimized for learning **architecture, modules, and data flow** quickly enough to make your first useful contribution.

## Companion docs

- `architecture-overview.md` - visual map of the main PostHog runtime pieces
- `data-flows.md` - concrete request and ingestion flow walkthroughs

## 1. The mental model

PostHog is a large monorepo with three architectural styles living side by side:

| Area | What it is | Why it matters |
| --- | --- | --- |
| `posthog/` | Legacy Django monolith | Still owns a lot of core API, models, ClickHouse access, HogQL, and background tasks |
| `products/` | Newer vertical slices | The direction of travel for product code: backend + frontend + manifest in one place |
| `nodejs/`, `rust/`, `services/` | High-throughput or standalone services | Ingestion, feature flags, capture, workers, and service-style components |

The big shift to understand is this:
**newer code tries to isolate products; older code is still centralized.**
You will read both styles in real contributions.

## 2. Start with these anchor files

Read these in order:

1. `docs/internal/monorepo-layout.md`  
   Best high-level map of the repo.
2. `products/README.md`  
   Explains the vertical-slice product layout.
3. `products/architecture.md`  
   Explains facades, contracts, logic, presentation, and isolation rules.
4. `posthog/settings/web.py`  
   Shows how product backend apps are registered in Django.
5. `posthog/api/__init__.py`  
   Shows how API routes from core and products are wired together.
6. `docs/published/handbook/engineering/project-structure.md`  
   Good plain-English summary of the major directories.
7. `CONTRIBUTING.md`  
   Sets expectations for external PRs.

## 3. How the repo is organized

### Core modules

| Path | Role |
| --- | --- |
| `frontend/` | React + TypeScript web app, mostly organized around scenes and Kea logic |
| `posthog/api/` | Django REST endpoints and route registration |
| `posthog/models/` | Core PostgreSQL-backed Django models |
| `posthog/clickhouse/` | ClickHouse schema and query-layer code |
| `posthog/hogql/` | HogQL compiler/execution layer |
| `posthog/tasks/` | Celery jobs and async backend work |
| `products/*` | Product-owned backend/frontend slices |
| `nodejs/` | Event ingestion, plugins, webhook/data-pipeline work |
| `rust/` | Performance-sensitive services like capture and feature flags |
| `common/hogli/` | Developer CLI and local workflow tooling |

### Product slices

The intended product shape is:

```text
products/<name>/
  backend/
    logic.py
    facade/
      api.py
      contracts.py
    presentation/
      serializers.py
      views.py
  frontend/
  manifest.tsx
```

That separation means:

- `facade/` is the public boundary
- `logic.py` holds business rules
- `presentation/` handles HTTP and serializers
- `manifest.tsx` wires frontend routes/scenes/navigation

Important caveat:
**not every product is there yet.**
You will still find older layouts.
For example, `products/feature_flags/` is closer to the older style, while `products/notifications/` reflects the newer split more clearly.

## 4. The three data flows you should learn first

### Flow A: frontend feature request path

Use this to understand a user-facing feature end to end.

```text
frontend scene / product manifest
  -> browser request
  -> Django router in posthog/api/__init__.py
  -> product presentation/views.py or legacy posthog/api/*
  -> logic.py / facade
  -> Postgres or ClickHouse
  -> serializer / response
  -> frontend renders result
```

**Concrete files to trace**

- `products/feature_flags/manifest.tsx`
- `posthog/api/__init__.py`
- `products/notifications/backend/presentation/views.py`
- `products/notifications/backend/logic.py`

### Flow B: modern isolated product path

Use Notifications as the cleaner example of the newer architecture.

```text
contract dataclass
  -> facade export
  -> logic implementation
  -> model write
  -> side effects on commit
  -> presentation layer exposes reads/actions
```

**Concrete files to trace**

1. `products/notifications/backend/facade/contracts.py`  
   Public input shape (`NotificationData`).
2. `products/notifications/backend/facade/api.py`  
   Public surface exports.
3. `products/notifications/backend/logic.py`  
   Real work: team lookup, flag gate, recipient resolution, DB write, Kafka publish.
4. `products/notifications/backend/presentation/views.py`  
   HTTP endpoints for list, unread count, mark read/unread.

This is the cleanest single example for learning:

- contract boundary
- logic boundary
- presentation boundary
- side effects after DB commit

### Flow C: event ingestion path

Use this to understand how PostHog handles high-volume event data.

```text
SDK/event source
  -> capture/ingestion service
  -> Kafka message
  -> joined ingestion pipeline
  -> preprocessing + team lookup
  -> group by token:distinct_id
  -> concurrent per-entity processing
  -> results + side effects
  -> storage / downstream outputs
```

**Concrete files to trace**

- `docs/published/handbook/engineering/project-structure.md`
- `nodejs/src/ingestion/analytics/joined-ingestion-pipeline.ts`

What to notice in `joined-ingestion-pipeline.ts`:

- `messageAware(...)` wraps the pipeline
- pre-team preprocessing happens first
- `teamAware(...)` gates team-scoped work
- events are grouped by `token:distinct_id`
- each group is processed concurrently, but events within a group are sequential
- `handleResults(...)` and `handleSideEffects(...)` are explicit steps

That file is one of the best real maps for the ingestion side of PostHog.

## 5. A practical onboarding sequence

### Phase 1: build the map

Goal: know where things live without understanding every implementation detail.

Do this:

1. Read the 7 anchor files above.
2. Write your own one-page summary with:
   - what still lives in `posthog/`
   - what lives in `products/`
   - what lives in `nodejs/` and `rust/`
3. Memorize the two key registration points:
   - `posthog/settings/web.py`
   - `posthog/api/__init__.py`

Exit criterion:
You can explain where a new Django app, route, or product feature would likely go.

### Phase 2: trace one request and one ingestion flow

Goal: understand runtime movement, not just directory names.

Do this:

1. Trace a browser/API path using Notifications.
2. Trace the ingestion builder chain in `joined-ingestion-pipeline.ts`.
3. Draw both flows in plain text.

Exit criterion:
You can explain the difference between:

- a product HTTP flow
- an ingestion pipeline flow
- a legacy monolith path

### Phase 3: set up your contributor workflow

Learn these commands early:

```bash
./bin/start
hogli test <path>
hogli test --changed
ruff check . --fix
ruff format .
pnpm --filter=@posthog/frontend typescript:check
pnpm --filter=@posthog/frontend build
```

Useful rule of thumb:

- backend tests: `hogli test`
- Python formatting/linting: `ruff`
- frontend type safety: frontend TypeScript check

### Phase 4: choose your first contribution type

Best first contributions, in order:

1. small docs mismatch in an area you just traced
2. serializer/view cleanup in a product with good boundaries
3. test-only fix
4. small UI polish in a clearly owned product
5. narrowly scoped bug fix with an existing issue

Avoid starting with:

- deep ingestion changes
- large cross-product refactors
- paid-feature work
- anything touching many services before you know the boundaries

## 6. How to read code efficiently in this repo

When you pick any issue, inspect it in this order:

1. **Entry point**  
   Route, manifest, signal, task, consumer, or command.
2. **Boundary**  
   Is this core monolith code, a product slice, Node ingestion, or Rust?
3. **Business logic**  
   Look for `logic.py`, pipeline steps, or service-layer code.
4. **Persistence/output**  
   Postgres model, ClickHouse query path, Kafka output, or async task.
5. **Tests**  
   Check nearby tests before changing anything.

If a feature touches `products/`, always ask:

- Is there a facade already?
- Is there a presentation layer already?
- Is this product isolated, or still mixed with legacy code?

## 7. Suggested study path by interest

| If you like... | Start here |
| --- | --- |
| Backend APIs | `posthog/api/`, `products/*/backend/presentation/`, serializers/viewsets |
| Product architecture | `products/README.md`, `products/architecture.md`, `products/notifications/backend/*` |
| Data infrastructure | `nodejs/src/ingestion/analytics/joined-ingestion-pipeline.ts`, `posthog/clickhouse/`, `posthog/hogql/` |
| Frontend | `frontend/src/scenes/`, `frontend/src/lib/`, product `manifest.tsx` files |
| Contribution workflow | `CONTRIBUTING.md`, `AGENTS.md`, `common/hogli/` |

## 8. A realistic first-contribution strategy

Use this filter:

1. Pick an issue in a product area, not a vague monolith-wide issue.
2. Prefer something with an obvious owner and a small blast radius.
3. Read the local tests before the implementation.
4. Match existing patterns exactly.
5. Keep your PR small and link the issue.

In PostHog, small and well-aligned usually beats ambitious.

## 9. What good understanding looks like

You are ready for meaningful contributions when you can answer these:

1. When should code go in `posthog/` vs `products/`?
2. How are product backend apps registered with Django?
3. How are API routes registered?
4. What is the role of `facade/contracts.py`, `facade/api.py`, `logic.py`, and `presentation/views.py`?
5. How does the ingestion pipeline structure work at a high level?
6. Which storage/query paths are PostgreSQL-oriented vs ClickHouse-oriented?

If you can answer those from memory, you are past the lost in the repo stage.

## 10. Recommended next reads after this guide

1. `products/visual_review/backend/*` for another modern product example
2. `frontend/src/scenes/urls.ts` to understand frontend route helpers
3. `posthog/api/mixins.py` and serializer-heavy viewsets to understand API patterns
4. `posthog/hogql/` if you want to understand query execution
5. Nearby tests in whatever area you choose to contribute to first
