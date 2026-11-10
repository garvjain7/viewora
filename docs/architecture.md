# 02 — Architecture overview (HLD)

> **Working draft, not a final architecture.** This describes a candidate direction assembled from current planning notes. Requirements and stack choices may change. ADR references below came from source notes; ADR documents are not currently present in this workspace, so those references do not prove that a decision was reviewed or finalized. The requirements document is [requirements.md](requirements.md); stack status and unresolved questions are in [stack.md](stack.md) and [open-questions.md](open-questions.md).

Scope: describe candidate boundaries and data flows, not imply that they are implemented. This document is organised by concern and module; P/L/S labels are planning gates, not delivery commitments.

## 1. Shape

A **modular-monolith candidate** with a Python API, a separately deployable storefront SSR process, static dashboard assets, background workers, PostgreSQL, Redis-compatible services, and object storage + CDN. This is one repository with multiple runtime/build artifacts, not one deploy artifact. The number of Redis instances and the queue/stream technologies remain open; see §5–6 and [stack.md](stack.md).

```
 Shopper phone ──► CDN (Cloudflare) ──► Storefront SSR ──┐
        │              │  assets (R2)                     │
        │              └──────────────────────────────────┤
 Merchant ─► Dashboard SPA ──► API (FastAPI) ◄────────────┘
                                 │
        ┌────────────┬───────────┼─────────────┬───────────────┐
        ▼            ▼           ▼             ▼               ▼
   Postgres     Redis-compatible services  Object store     Outbound
   (durable     (cache + candidate short-   (R2 candidate:  (Razorpay,
   domain data, lived state/streams;         originals,       SMS, email,
   outbox,      durability/separation open) derived, USDZ,   AI/try-on
   vector)                                   posters)         providers)
        ▲                          │
        └──── candidate workers: asset, gen, analytics, default
```

The shopper-facing SSR server is currently planned with React Router v7 and Node, provisionally. It renders pages and calls the Python API; business rules and database access stay in the Python backend unless a later design explicitly changes that boundary. The dashboard is currently planned as a Vite SPA. Neither web app is implemented yet.

## 2. Modules

Candidate module boundaries: each module owns its domain behavior and data access, exposes an explicit interface, and communicates through events where an immediate response is not needed. These boundaries are a design goal, not enforced yet; the referenced `conventions.md` is not present.

| Module | Owns | Reads from others via | Notes |
|---|---|---|---|
| platform | config, db, cache, outbox, jobs, auth primitives, errors, telemetry | — | shared kernel; no business rules |
| identity | merchants (tenants), users, roles, sessions | — | tenant context origin |
| billing | plans, subscriptions, entitlements, credit ledger | identity | single source of limits (BIL-03) |
| catalog | products, variants, categories, vertical profiles, slugs | identity, billing | writes `product.changed` events |
| inventory | stock, reservations | catalog | **only** module that writes stock |
| assets | uploads, 3D/2D pipeline, derived files | catalog | GLB bytes never via API |
| qr | codes, targets, scan resolve | catalog | Redis destination cache is a candidate; resolver boundary is open |
| storefront | page composition, SEO, viewer wiring | catalog, inventory, reviews, discovery | read-only |
| cart | carts | catalog, inventory | Redis-backed temporary cart is a candidate; recovery/loss behavior is open |
| orders | orders, state machine | cart, inventory, payments | writes money state |
| payments | Razorpay orders, webhooks, reconciliation, refunds | orders | **only** third-party call in request path |
| search | indexes (FTS/trigram), query API | catalog | engine is swappable (ADR-0004) |
| reviews | ratings, moderation, aggregates | orders | |
| analytics | event ingest, rollups, dashboards | all (events) | |
| discovery | `discovery_items`, embeddings, similar lists | catalog, inventory (events) | cross-tenant projection (ADR-0016) |
| notifications | email/SMS send + log | any (events) | |
| fulfillment | shipping/pickup records, port to delivery platform | orders | |
| ai | gateway, comparison, page analysis, generation orchestration | catalog, assets, analytics | no LLM in request critical path |
| admin | console, moderation, audit | all (read), few writes | |

Rules that make this a monolith rather than a ball of mud: only the public interface of another module is imported (lint-enforced, MNT-01); modules communicate asynchronously through events when the other side does not need an immediate answer; only `inventory`, `orders`, `payments`, `billing` write money or stock state.

## 3. Process types

| Process | Entrypoint | Scales on | Notes |
|---|---|---|---|
| api | FastAPI/uvicorn (Python) | request rate | Candidate public/business API; not implemented |
| storefront | React Router v7 SSR (Node; provisional) | request rate | Renders shopper pages and calls Python API; no direct DB access intended |
| dashboard | React/Vite SPA (candidate), served as static files | — | No production Node server required solely to serve static output |
| worker_default | Python worker; queue technology open | queue depth | email, SMS, rollups, webhooks |
| worker_asset | Python worker; queue technology open | queue depth | isolated asset processing; sandbox requirements need implementation |
| worker_gen | Python worker; queue technology open | work backlog/cost | AI generation orchestration |
| worker_analytics | consumer for selected event transport | lag/backlog | batch ingest to Postgres; Redis Streams is a candidate |
| scheduler | periodic job mechanism open | — | reconciliation, retention, discovery refresh |

Procrastinate is present in backend dependencies and is a PostgreSQL-backed queue candidate. Redis-backed Celery/Arq are alternatives, not currently selected. Do not describe Redis as the worker broker until that decision is made.

## 4. Data and tenancy

- **Postgres** is the system of record. Multi-tenant, shared schema, `tenant_id` on every tenant table, **RLS enforced** with `FORCE ROW LEVEL SECURITY`, application connects as a non-owner role, tenant set per transaction with `set_config('app.tenant_id', …, true)`. (ADR-0003, SEC-01)
- IDs: UUIDv7. Money: integer paise. (ADR-0006)
- Flexible product data: `attributes` JSONB validated against per-vertical profile JSON Schema, GIN indexed for filters. No document DB. (ADR-0004)
- Cross-tenant discovery uses a **separate public projection**, `discovery_items`, with no RLS, a single writer, and a read-only role for readers; no tenant-private field is ever copied in. (ADR-0016, DSC-05)
- Connection pooling, including whether to use PgBouncer and which workloads connect directly, needs validation against the chosen PostgreSQL host and queue.
- Migration direction: Alembic/SQL migrations are candidates. The current workspace has no migration revisions or generator. The owner wants root `schema.sql` as a generated full-database snapshot for inspection and AI context; it must not be hand-edited. The actual migration source and generation workflow remain to be implemented.

## 5. Read path: product page and QR scan

```
scan → /q/{code}
   resolver boundary OPEN (API or edge handler)
   → cache candidate (qr:{code} → current target)
   → database fallback on miss / invalidation gap
   → 302 /{merchant}/{product}
   → scan event through selected telemetry path

/{merchant}/{product}
   CDN (short TTL + stale-while-revalidate, keyed by versioned URL)
   → SSR: product aggregate (cache) + overlay (price/stock/rating, separate short TTL)
   → HTML with poster image inline; viewer JS lazy
   → client: fetch LOD0 GLB from CDN (immutable hash URL), KTX2 textures
   → client: similar-products fetch (post-LCP)
```

Candidate cache layers: browser/CDN, optional process-local cache, Redis-compatible cache, and PostgreSQL fallback. A cache hit is a performance optimization, not a correctness guarantee.
- QR code identity can be immutable while its target changes. Cache `code → current target`, and define target/slug-change invalidation or a safe TTL.
- Candidate cache techniques include versioned keys, request coalescing on misses, TTL jitter, and short negative caching; validate complexity against measured traffic.
- **Read-model split candidate:** relatively stable product data (title, media, attributes, viewer configuration) can cache separately from volatile price, stock, and rating data.
- Checkout must re-read price and inventory from PostgreSQL; cached data is never authoritative.
- Cache and state are logical workload roles. Separate physical instances may isolate eviction and failure risk; a shared service costs less but can couple cache pressure to state. Choose using budget, provider limits, and accepted recovery behavior. A temporary/free service is not presumed durable.
- Additional Redis candidates include rate-limit counters, OTPs and other short-lived tokens, temporary idempotency coordination, carts, telemetry streams, live job progress, and an availability bitmap. Each needs a defined TTL, loss/rebuild behavior, and database authority where applicable. Do not add every candidate by default.

## 6. Write path and events

Durable domain changes should commit with an outbox record in one PostgreSQL transaction. How a durable background job is atomically enqueued depends on the queue choice: a PostgreSQL-backed queue may join the transaction; a Redis-backed queue needs an outbox/relay or equivalent recovery mechanism. The queue is not selected yet.

| Event class | Transport | Examples |
|---|---|---|
| Money/state (must not be lost) | Postgres outbox → relay | order.paid, stock.changed, subscription.changed, product.changed |
| High-volume telemetry | Redis Streams is a candidate; transport and durability are open | scan, view, 3d_interaction, ar_launch |

Consumers should tolerate retries and duplicates. Define retention, backpressure, replay, and what “without dropping” means before selecting a stream transport. PostgreSQL analytics ingestion should be idempotent. Kafka/Redpanda are not part of the initial working direction; revisit only against a measured need.

## 7. Assets and 3D pipeline

1. Merchant requests a signed upload URL; file goes straight to object storage (AST-01).
2. Job enqueued transactionally; sandboxed worker (no network, CPU/mem/time limits) validates type, runs `gltf-transform`/`gltfpack`/`toktx`, produces meshopt + KTX2 LOD0/LOD1, poster, thumbnail, USDZ; checks scale against declared dimensions; enforces triangle/texture budgets (AST-02, AST-03, AST-06).
3. Derived files are written at content-hash paths on R2 and served via Cloudflare with immutable cache headers (ADR-0012).
4. 2D images: libvips → AVIF/WebP/srcset (AST-05).
5. AI generation and assisted digitization feed the same pipeline as merchant uploads, followed by a QA gate (ADR-0015).

The durable asset/job status belongs in recoverable storage (the schema snapshot has asset states and AI generation job states). A Redis progress channel may support live dashboard updates, but it should not be the sole record of completion or failure. Percentage progress and WebSocket/SSE delivery are not yet requirements.

Client-side budget rules (PERF-04, PERF-05, PERF-11): poster first; viewer runtime lazy-loaded; decoders (Draco/meshopt/KTX2/MediaPipe WASM) self-hosted; connection-aware LOD choice; no new storefront dependency above 20 KB gzipped without an ADR.

## 8. Try-on

- **Glasses:** on-device only. MediaPipe face landmarks (WASM), One-Euro filter, head occluder mesh, scale from pupillary distance, per-product metadata (`product_tryon_metadata`). Three.js, lazy-loaded. No frames leave the device (TRY-01, TRY-02).
- **Clothing:** hosted VTON provider behind a `TryOnProvider` interface; async job, credit-metered, explicit consent, photos deleted ≤24 h (TRY-03, TRY-04). Size recommendation is built in-house.
- Earrings/necklaces later on-device; rings and footwear are out of scope. (ADR-0014)

## 9. Discovery: similar products

Offline, per category family, never a full-catalog scan at request time.

1. `discovery_items` projection is fed by `product.changed` and stock events.
2. A CPU worker computes embeddings with a pretrained, commercially licensed model (Marqo FashionSigLIP / Ecommerce, Apache 2.0) for image + text; structured attributes (color, type) are stored beside.
3. pgvector HNSW indexes are **list-partitioned by category family** (pgvector ≥0.8 iterative scan). Candidate retrieval is within the family, then rerank with attribute match and MMR diversity, ≤2 per merchant, near-duplicate clusters collapsed.
4. Precomputed neighbour lists live in the discovery read model. A Redis bitmap is a candidate fast availability filter using `product_seq`; it is derived from inventory and may be briefly stale. Define update/rebuild behavior. Checkout always uses PostgreSQL inventory.
5. Same-merchant first, then cross-merchant opt-in merchants; click-out to the other merchant's Viewora storefront page.
6. Holdout group measures real lift (DSC-08).

The cross-merchant click-out rule is **Proposed**, not final. Confirm whether cross-merchant discovery links to the other storefront or supports another journey; no combined-cart behavior should be inferred. See [open questions](open-questions.md).

## 10. Search

Postgres full-text (`tsvector`, weighted) + `pg_trgm` for typos and autocomplete, filters through indexed columns/GIN. Scoped to merchant or global. The engine boundary is a `SearchIndex` port so replacement (OpenSearch/Typesense) is possible when a trigger in ADR-0004 fires (e.g. p95 > 150 ms at representative size after tuning). (SRC-*)

## 11. Auth and security

Authentication and authorization are owned by the Python application. OTPs and other short-lived verification values may use TTL storage. Refresh-token handling is unresolved: the schema snapshot models hashed rotating tokens in PostgreSQL, while older architecture notes mention Redis revocation state. Choose one design based on revocation, reuse detection, and recovery needs; do not assume JWT refresh tokens or an all-Redis allowlist. Redis may speed rate limiting or temporary coordination, but durable webhook/idempotency correctness must survive Redis loss. External AI providers receive only data intentionally sent by application services; they get no direct database credentials. MCP is an optional interoperability layer, not a permission system or database boundary. See [open-questions.md](open-questions.md).

## 12. Payments and orders

Checkout: re-price and reserve stock in Postgres, create Razorpay order (with timeout; the only third-party call in the request path), hand off to hosted checkout, finalize on verified webhook. Order state machine is persisted; webhooks tolerate replay and reorder; daily reconciliation. Split settlement is **OPEN** (PAY-05).

## 13. AI

The product brief intends AI to help shoppers make product decisions as well as help merchants improve product information. Candidate capabilities include natural-language product research, comparisons, recommendations, and merchant-page analysis. Define the assistant's interaction model, personalization, tool/action permissions, evaluation, and retention before implementing it. Route model calls through Viewora-owned services with budgets, timeouts, logging, and provider-specific data controls; do not expose raw database access to an external LLM. MCP may be evaluated for reusable capability/context interoperability, but it is not required and does not replace Viewora authorization. Keep model calls out of checkout and latency-critical page rendering. Independent Rufus research is useful for evaluating multi-turn task success and robustness across writing styles, not as a source for Rufus's private architecture or a template to copy.

## 14. Infrastructure

| Gate | Deployment |
|---|---|
| Initial local work | No Docker Compose per owner instruction. Use local Python/Node toolchains and selected hosted services where credentials and limits permit. |
| Current provider direction | Neon PostgreSQL, Render Redis-compatible service, Cloudflare R2/CDN. These are working provider choices, not immutable decisions; validate database extensions, Redis persistence/eviction, regions, quotas, and cost before relying on them. |
| Deployment topology | Not decided. Keep deployables separable (Python API/workers, Node SSR if retained, static dashboard); choose hosting and scaling from actual needs and budget. |

Observability: structured logs (structlog), OpenTelemetry traces, RED metrics, Sentry, RUM for Web Vitals and 3D/try-on marks (OBS-*). SMS goes through an Indian provider; DLT registration has lead time and starts early (CMP-04).

## 15. Failure and degradation

Candidate degradation order: disable AI, try-on, and similar-product enhancements before core product pages and checkout. CDN or Redis may preserve selected hot reads during a database outage only if cache freshness and failure behavior are designed and measured. Cache loss should cost latency, not correctness. State-Redis loss may lose or interrupt carts, rate limits, or telemetry unless persistence/recovery is configured; acceptable loss and restore behavior remain open. Payments, inventory, and durable idempotency must remain correct without Redis.

## 16. What would make us change this

Every ADR carries a measurable *Revisit when* line. The architecture is intentionally revisable at those seams: the outbox (to Kafka), the `SearchIndex` port, the `TryOnProvider` port, the `Fulfillment` port, the AI gateway, and the process-type boundaries (to separate services).
