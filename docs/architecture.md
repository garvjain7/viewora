# 02 — Architecture overview (HLD)

Scope: how Viewora is put together. Requirements are in [01](01-requirements.md); every non-obvious choice below links to an ADR. This document is organised by **concern and module**, never by time. How a deployment grows is described by **gates** (P/L/S) and measurable triggers.

## 1. Shape

A **modular monolith** with separate **process types**, one Postgres, two Redis instances, object storage + CDN, and a few sandboxed workers. One codebase, one deploy artifact, several entrypoints. (ADR-0002)

```
 Shopper phone ──► CDN (Cloudflare) ──► Storefront SSR ──┐
        │              │  assets (R2)                     │
        │              └──────────────────────────────────┤
 Merchant ─► Dashboard SPA ──► API (FastAPI) ◄────────────┘
                                 │
        ┌────────────┬───────────┼─────────────┬───────────────┐
        ▼            ▼           ▼             ▼               ▼
   Postgres     Redis cache   Redis state   Object store     Outbound
   (RLS,        (no persist,  (AOF+replica: (R2: originals,  (Razorpay,
   pgvector,    LFU)          streams,      derived, USDZ,   SMS, email,
   outbox,                    limits, carts, posters)         AI/try-on
   jobs)                      locks, idem.)                   providers)
        ▲                          │
        └──── workers: asset, gen, analytics, default ◄── Procrastinate + Streams
```

## 2. Modules

Each module owns its tables, exposes a public service interface, emits events, and may not be imported around. (conventions.md has the exact folder layout.)

| Module | Owns | Reads from others via | Notes |
|---|---|---|---|
| platform | config, db, cache, outbox, jobs, auth primitives, errors, telemetry | — | shared kernel; no business rules |
| identity | merchants (tenants), users, roles, sessions | — | tenant context origin |
| billing | plans, subscriptions, entitlements, credit ledger | identity | single source of limits (BIL-03) |
| catalog | products, variants, categories, vertical profiles, slugs | identity, billing | writes `product.changed` events |
| inventory | stock, reservations | catalog | **only** module that writes stock |
| assets | uploads, 3D/2D pipeline, derived files | catalog | GLB bytes never via API |
| qr | codes, targets, scan resolve | catalog | hot path, Redis-first |
| storefront | page composition, SEO, viewer wiring | catalog, inventory, reviews, discovery | read-only |
| cart | carts | catalog, inventory | state Redis |
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
| api | FastAPI/uvicorn | CPU / request rate | stateless, ≥2 tasks (AVL-04) |
| storefront | React Router v7 SSR (provisional, ADR-0011) | request rate | talks to API internally; cache-friendly |
| dashboard | static Vite SPA on CDN | — | |
| worker_default | Procrastinate | queue depth | email, SMS, rollups, webhooks |
| worker_asset | Procrastinate, sandboxed | queue depth, to zero | gltf-transform, gltfpack, toktx, libvips; no network |
| worker_gen | Procrastinate | credits queue, to zero | AI generation orchestration |
| worker_analytics | Streams consumer group | stream lag | batch `COPY` to Postgres |
| scheduler | Procrastinate periodic | — | reconciliation, retention, discovery refresh |

## 4. Data and tenancy

- **Postgres** is the system of record. Multi-tenant, shared schema, `tenant_id` on every tenant table, **RLS enforced** with `FORCE ROW LEVEL SECURITY`, application connects as a non-owner role, tenant set per transaction with `set_config('app.tenant_id', …, true)`. (ADR-0003, SEC-01)
- IDs: UUIDv7. Money: integer paise. (ADR-0006)
- Flexible product data: `attributes` JSONB validated against per-vertical profile JSON Schema, GIN indexed for filters. No document DB. (ADR-0004)
- Cross-tenant discovery uses a **separate public projection**, `discovery_items`, with no RLS, a single writer, and a read-only role for readers; no tenant-private field is ever copied in. (ADR-0016, DSC-05)
- Connections: PgBouncer (transaction mode, ≥1.21 for prepared statements) for API; workers connect directly because they use LISTEN/NOTIFY. (ADR-0010)
- Migrations: hand-written SQL via Alembic, expand/contract; CI regenerates `schema.sql`.

## 5. Read path: product page and QR scan

```
scan → /q/{code}
   CDN? miss → API/edge handler
   L1 in-process LRU → Redis cache (qr:{code} → target)  [ <10 ms ]
   → 302 /{merchant}/{product}   (event: scan → state Redis Stream)

/{merchant}/{product}
   CDN (short TTL + stale-while-revalidate, keyed by versioned URL)
   → SSR: product aggregate (cache) + overlay (price/stock/rating, separate short TTL)
   → HTML with poster image inline; viewer JS lazy
   → client: fetch LOD0 GLB from CDN (immutable hash URL), KTX2 textures
   → client: similar-products fetch (post-LCP)
```

Caching layers (outermost to innermost): browser/CDN, in-process L1, Redis L2, Postgres. Rules:
- Versioned keys and epoch counters instead of delete-and-hope invalidation.
- Singleflight on misses; TTL jitter; negative caching for unknown slugs/codes.
- **Overlay split:** the product aggregate (title, media, attributes) changes rarely; price, stock, and rating are a separate small overlay with its own TTL and event-driven refresh.
- Checkout never trusts cache (CRT-02, COR-01).
- Two Redis instances so a cache flush or eviction cannot take down streams, carts, rate limits or locks. (ADR-0007)

## 6. Write path and events

State change and its side effects commit in one Postgres transaction: domain row + **outbox** row + Procrastinate job row where needed. A relay publishes outbox rows to consumers. (ADR-0009, COR-05)

| Event class | Transport | Examples |
|---|---|---|
| Money/state (must not be lost) | Postgres outbox → relay | order.paid, stock.changed, subscription.changed, product.changed |
| High-volume telemetry | Redis Streams, sharded by tenant hash, consumer groups, XAUTOCLAIM, DLQ, 24 h retention | scan, view, 3d_interaction, ar_launch |

Consumers are idempotent; at-least-once is assumed (COR-04). The analytics sink uses idempotent `COPY` batches. Kafka/Redpanda is **not** adopted; the triggers that would change that are in ADR-0008. (ADR-0008)

## 7. Assets and 3D pipeline

1. Merchant requests a signed upload URL; file goes straight to object storage (AST-01).
2. Job enqueued transactionally; sandboxed worker (no network, CPU/mem/time limits) validates type, runs `gltf-transform`/`gltfpack`/`toktx`, produces meshopt + KTX2 LOD0/LOD1, poster, thumbnail, USDZ; checks scale against declared dimensions; enforces triangle/texture budgets (AST-02, AST-03, AST-06).
3. Derived files are written at content-hash paths on R2 and served via Cloudflare with immutable cache headers (ADR-0012).
4. 2D images: libvips → AVIF/WebP/srcset (AST-05).
5. AI generation and assisted digitization feed the same pipeline as merchant uploads, followed by a QA gate (ADR-0015).

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
4. Precomputed neighbour lists per item live in Redis/Postgres; at render, filter by the Redis **in-stock bitmap** (dense `product_seq`, SETBIT/GETBIT).
5. Same-merchant first, then cross-merchant opt-in merchants; click-out to the other merchant's Viewora storefront page.
6. Holdout group measures real lift (DSC-08).

The cross-merchant click-out rule is **Proposed** until the owner confirms it (ADR-0016, 05 A-02).

## 10. Search

Postgres full-text (`tsvector`, weighted) + `pg_trgm` for typos and autocomplete, filters through indexed columns/GIN. Scoped to merchant or global. The engine boundary is a `SearchIndex` port so replacement (OpenSearch/Typesense) is possible when a trigger in ADR-0004 fires (e.g. p95 > 150 ms at representative size after tuning). (SRC-*)

## 11. Auth and security

Short-lived access tokens, revocable refresh tokens (revocation list in state Redis). Roles per merchant. RLS is the backstop for application bugs. Razorpay webhooks verified by signature, processed idempotently. Uploads sandboxed. Admin 2FA, audit log. See SEC-*, PRV-*.

## 12. Payments and orders

Checkout: re-price and reserve stock in Postgres, create Razorpay order (with timeout; the only third-party call in the request path), hand off to hosted checkout, finalize on verified webhook. Order state machine is persisted; webhooks tolerate replay and reorder; daily reconciliation. Split settlement is **OPEN** (PAY-05).

## 13. AI

One gateway module for every model call (LLM, generation, embeddings) with budget caps, timeouts, retries, fallback, logging and input-hash caching. No AI call is in page render or checkout (AIX-05). Provider swap is contained to the gateway (AI-04).

## 14. Infrastructure

| Gate | Deployment |
|---|---|
| P | Docker Compose on one India VPS: api, storefront, workers, Postgres, 2× Redis, Caddy. Cloudflare in front. Backups to R2. |
| L | AWS Mumbai (ap-south-1): ECS Fargate (API/SSR ≥2 tasks over 2 AZs; workers scale on queue backlog; asset/gen to zero), managed Postgres and Redis, R2 + Cloudflare, Terraform, GitHub Actions. (ADR-0013, provisional; hosting provider also depends on budget/credits, 05 A-06) |
| S | Kubernetes (EKS + KEDA) in a **scale lab** first, adopted only when a trigger in ADR-0013 fires. Kafka, a dedicated search engine, ClickHouse, Cassandra, MongoDB: excluded now; each has an adoption trigger in its ADR. |

Observability: structured logs (structlog), OpenTelemetry traces, RED metrics, Sentry, RUM for Web Vitals and 3D/try-on marks (OBS-*). SMS goes through an Indian provider; DLT registration has lead time and starts early (CMP-04).

## 15. Failure and degradation

Planned order of shedding (AVL-05): AI → try-on → similar products → 3D (poster only) → everything except checkout read/write last. Hot content survives a primary-DB outage for ≥5 min through CDN + Redis (SCAL-03). A redis-cache loss costs latency, not correctness (COR-07). A state-Redis loss is a recoverable incident: carts restore from the last persisted state, streams from AOF, idempotency keys fall back to Postgres unique constraints.

## 16. What would make us change this

Every ADR carries a measurable *Revisit when* line. The architecture is intentionally revisable at those seams: the outbox (to Kafka), the `SearchIndex` port, the `TryOnProvider` port, the `Fulfillment` port, the AI gateway, and the process-type boundaries (to separate services).