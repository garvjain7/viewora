# 03 — Data model

## Status and evidence

This document describes the current `schema.sql` snapshot as a starting point for discussion. It is not a record of final product or architecture decisions. Comments and rationale copied from the source material have not all been independently verified; treat them as hypotheses unless confirmed by the owner or implementation.

In this workspace, `schema.sql` exists, but `backend/migrations/versions/` contains only `.gitkeep`; there is no migration history that can reproduce this snapshot. `tools/` also contains only `.gitkeep`, so the referenced schema-generation and schema-test scripts are not present here. Therefore the claims that the snapshot is generated from migrations, that CI checks for drift, or that 79 schema assertions pass are not verified and should not be relied on. The owner wants to retain a complete generated snapshot for easy AI inspection; the generation workflow still needs to be selected and built.

The DDL uses PostgreSQL features including column-list `ON DELETE SET NULL` and the `vector` extension. The exact minimum supported PostgreSQL and pgvector versions, and whether this DDL has been applied successfully to any database, are not established by the files currently in this workspace.

## 1. Current schema conventions (not locked decisions)

1. **One Postgres schema per module.** `identity`, `billing`, `assets`, `catalog`, `inventory`, `qr`, `analytics`, `ai`, `discovery`, `orders`, `payments`, `fulfillment`, `reviews`, `notifications`, plus `platform` (kernel) and `ref` (reference data). The schema name tells you who may write to it.
2. **Tenant = merchant.** Every tenant table carries `tenant_id`, with `FORCE ROW LEVEL SECURITY` and the policy `tenant_id = platform.app_tenant_id()`. The application role is not the owner and has `NOBYPASSRLS`. No role bypasses RLS; admins act inside a tenant context (audit-logged).
3. **Composite foreign keys inside a tenant.** References between tenant tables are `(tenant_id, id) → (tenant_id, id)`. A row cannot point at another tenant's row even if application code tries. (ADR-0017)
4. **Money is `bigint` paise, currency fixed to INR by CHECK.** IDs are UUIDv7. Timestamps are `timestamptz`.
5. **Status columns are `text` + CHECK**, not enums, so adding a state is a cheap migration.
6. **Soft delete (`deleted_at`) for merchant-facing entities; hard delete only by retention jobs.** Partial unique indexes use `WHERE deleted_at IS NULL`.
7. **Append-only where history matters:** `audit_log`, `credit_ledger`, `stock_movements`, `status_history`. `UPDATE/DELETE/TRUNCATE` are revoked from the application role.
8. **Things that must outlive other things have no cascading FK to them:** QR codes outlive products (`ON DELETE SET NULL`), analytics events outlive everything (no FK), orders block hard deletion of ordered products (`NO ACTION`).
9. **Cross-tenant data lives only in a public projection** (`discovery.items`), with no RLS, one writer role, and no private fields. (ADR-0016)
10. **Candidate Redis uses:** short-lived OTP/reset state and carts may live in Redis with TTL. This is not a locked storage decision. Define loss/recovery and security properties for each value; durable identity, order, payment, and inventory facts stay in PostgreSQL. See [stack.md](stack.md).

## 2. Tables by module

| Schema | Tables | Notes |
|---|---|---|
| platform | `outbox`, `idempotency_keys`, `audit_log` | Snapshot tables only; exact Redis/database idempotency relationship is open. Outbox has no RLS in the snapshot. Functions: `app_tenant_id()`, `app_user_id()`, `uuid_v7()`, `enable_tenant_rls()` |
| ref | `vertical_profiles`, `categories` | Seeded with 7 verticals; categories hold the Google taxonomy id, path, and the discovery `family` |
| identity | `users`, `merchants`, `merchant_slugs`, `memberships`, `refresh_tokens`, `shopper_addresses`, `platform_roles` | One login → many merchants through `memberships`. Reserved slugs enforced by CHECK. Slug history by trigger |
| billing | `plans`, `subscriptions`, `credit_ledger`, `credit_balances` | One live subscription per merchant (partial unique). Ledger trigger keeps balance; overspend fails the CHECK |
| assets | `assets`, `derivatives` | Original + derived (lod0/lod1/poster/thumb/usdz/avif/webp) with content hash, triangles, texture size, pipeline version |
| catalog | `collections`, `products`, `product_slugs`, `product_collections`, `variants`, `product_media`, `product_tryon` | `products.search` is a generated weighted `tsvector` (`simple` config). `product_seq` is the dense id for the stock bitmap |
| inventory | `stock_levels`, `reservations`, `stock_movements` | `CHECK (unlimited OR reserved <= on_hand)`; reserve is one conditional `UPDATE` |
| qr | `codes`, `legacy_redirects` | `code` immutable (trigger). Targets nullable FKs with `SET NULL`. Lookup-by-code policy for the resolve path |
| analytics | `events` (monthly partitions), `daily_product_stats`, `daily_dimension_stats`, `experiment_assignments` | Partitions in `analytics_p`, unreachable by the app role except through the parent |
| ai | `outputs`, `generation_jobs`, `tryon_requests` | `outputs` unique on input hash. Try-on photos carry `purge_after` (24 h) |
| discovery | `items` (list-partitioned by category family), `neighbors` | No RLS. Written only by `viewora_discovery_writer`. Embedding column is `vector(768)` — **OPEN, see §7** |
| orders | `orders`, `order_items`, `status_history` | Totals enforced by CHECK. Idempotency key unique per merchant. Address and prices snapshotted |
| payments | `payments`, `refunds`, `webhook_events` | `webhook_events` is not tenant-scoped, keyed `(provider, event_id)` |
| fulfillment | `fulfillments` | `provider = 'manual'` now; last-mile platform later through the port |
| reviews | `reviews`, `reports`, `product_rating_stats`, `saved_products` | One review per purchased item (`UNIQUE(order_item_id)`). Wishlist is per user |
| notifications | `send_log`, `opt_outs`, `inbox` | Recipients stored as hashes |

53 tables in 16 schemas.

## 3. Resolving before the tenant is known

Three lookups must work before a tenant context exists. Each uses a narrow, public-data policy keyed by a transaction-local setting, rather than a bypass role:

| Lookup | Caller sets | Policy | Returns |
|---|---|---|---|
| Storefront `/{merchant}/…` | `app.merchant_slug` | `identity.merchants.lookup_by_slug` (SELECT) | merchant id, then set `app.tenant_id` |
| QR scan `/q/{code}` | `app.qr_code` | `qr.codes.lookup_by_code` (SELECT) | target path, status |
| Login | `app.user_id` | `identity.memberships.tenant_or_self` | the user's merchants and roles |

The source notes propose a Redis-first resolve path with these policies as a database fallback. This is a cache strategy candidate, not a property guaranteed by the schema. A QR code can remain immutable while its target changes, so target updates need cache invalidation or a bounded TTL. Whatever a lookup exposes is public data and must be reviewed as such.

## 4. Candidate hot paths and budgets (not measured yet)

| Path | Budget | Reads |
|---|---|---|
| QR resolve (miss) | 1 statement | `qr.codes` by `code` (unique index). `target_path` is denormalised so no join is needed; the qr module refreshes it from `catalog.product.slug_changed` events |
| Product page | ≤ 3 | product + default variant + media in one query each; price/stock/rating come from the overlay cache |
| Similar products | Candidate: 1 read-model lookup + bitmap filter | `discovery.neighbors` by `(family, product_id)`, then optional Redis `GETBIT` for availability. Bitmap is derived and may be stale; checkout inventory is authoritative in PostgreSQL. |
| Reserve stock | 1 per line | conditional `UPDATE inventory.stock_levels … WHERE unlimited OR on_hand − reserved >= $q`; zero rows updated = out of stock |
| Debit credits | 1 | insert into `credit_ledger`; trigger updates the balance; CHECK rejects overspend |
| Event ingest | batched `COPY` | into `analytics.events`; replay is harmless because the PK is `(occurred_at, event_id)` (producer must reuse the original `occurred_at`) |

## 5. Retention and partitioning

- `analytics.events`: monthly range partitions; the scheduler (as the owner role) calls `analytics.ensure_month_partition()` ahead of time; the default partition catches strays and is monitored. Raw retention bounded by dropping old partitions; rollups are kept.
- `discovery.items`: list-partitioned by category family; `discovery.ensure_family_partition(family)` creates a partition when a new family appears (owner role).
- `platform.idempotency_keys` and `identity.refresh_tokens`: purged by expiry job. `ai.tryon_requests`: photos/results deleted at `purge_after`.
- `platform.outbox`: published rows purged after a retention window.

## 6. ScanVista → Viewora mapping

The mapping below is historical context from the source notes, not a verified migration specification. The ScanVista repository, its `schema.sql`, and `server/migrate.js` are not present in this workspace, so the stated source, differences, and claims about which migration ran have not been checked. Do not use this table as a data-migration plan without comparing it to the actual prototype and confirming the intended Viewora behavior.

| ScanVista | Viewora | Change |
|---|---|---|
| `users` | `identity.users` | + `phone`; email nullable; status; `citext` email |
| — | `identity.merchants`, `memberships` | New: tenant and roles. See §7 on `projects` |
| `refresh_tokens` | `identity.refresh_tokens` | Adds `family_id`, `rotated_at`, `revoked_at` for rotation with reuse detection (ScanVista deleted all tokens on every login, logging out other devices) |
| `reset_password_tokens`, `two_factor_tokens` / `two_factor_codes` | State Redis (TTL) | Removed from Postgres |
| `projects` | **OPEN**: `identity.merchants` or `catalog.collections` | §7 |
| `products` | `catalog.products` + `variants` + `product_media` + `assets` | Split. `price NUMERIC` → `variants.price_paise = round(price × 100)`. `slug` global → unique per merchant with history. `features`/`specs` JSONB → `attributes` validated per vertical. `buy_url` dropped (checkout is in Viewora). `ai_summary`, `ai_use_cases`, generation status → `ai.outputs`, `ai.generation_jobs` |
| `qr_codes` | `qr.codes` | `qr_token` → `code`; `destination_url` → `target_path` + typed target; `scan_count` dropped (hot-row write on every scan); `qr_image_url` dropped (generated on demand). Cascade from product removed |
| — | `qr.legacy_redirects` | Printed ScanVista QRs encode `{host}/p/{slug}`, not the token. `/p/{slug}` must keep redirecting. Populated only if live data exists |
| `product_embeddings` (1536-d ada-002, one global HNSW) | `discovery.items.embedding` (partitioned) | New model and dimension; not migrated, recomputed |
| `qr_scans` | `analytics.events` (`type = 'scan'`) | Partitioned, no FK, salted rotating `session_hash` instead of plain IP hash |
| `user_interactions` | `analytics.events` | Events, not rows with FKs |
| `voice_queries` | Not represented in this snapshot | The schema does not currently define conversational assistant sessions or turns. This absence does not establish that conversation history is unwanted, ephemeral, or out of scope. Decide data capture, privacy, retention, and evaluation needs from Viewora's assistant requirements before designing these tables. |
| `comparison_sessions` | `ai.outputs` (`kind = 'comparison'`) | Cached by input hash |
| `product_favorites` (created in `migrate.js`) | `reviews.saved_products` | Per user, across merchants |
| `notifications` (created in `migrate.js`) | `notifications.inbox` | Tenant-scoped |
| soft-delete enum + `purge_at` | `deleted_at` + retention jobs | No cascade purge of QR or analytics |

## 7. Open items

| Item | Effect | Where tracked |
|---|---|---|
| What a ScanVista "project" is (a shop, or a folder) | Unresolved in the source notes; inspect the prototype and confirm intended Viewora behavior before mapping | Not verified; referenced planning files are not present in this workspace |
| Live ScanVista data / printed QRs | Whether such data exists and needs migration is unknown | Not verified; inspect the prototype/deployment and confirm with the owner |
| Embedding dimension | The DDL declares `vector(768)`, while its comment calls this dimension open. The correct dimension depends on the selected embedding model and is not confirmed | Confirm before loading embeddings |
| Category taxonomy seed | `ref.categories` is empty; load the Google product taxonomy and assign `family` | — |
| Vector DDL verification | The snapshot declares `vector(768)` and vector indexes. No migration or test harness is present in this workspace, so successful application and index behavior are unverified here | Confirm against the chosen PostgreSQL/pgvector environment |
| Not modelled | The snapshot has no explicit tables for promotions/coupons, returns/exchanges, product Q&A, or conversational assistant sessions/turns. Whether these belong in Viewora is a product decision, not established by their absence here | Requirements to be clarified |
| Tax | The snapshot has `orders.tax_paise`; tax and invoicing requirements are not established by this file | Requirements to be clarified |
| Partition maintenance role | The DDL defines partition helper functions. The deployment role and maintenance process are not implemented or verified in this workspace | Deployment design |
| Session setting `app.*` spoofing | The DDL uses transaction-local settings in RLS and lookup policies. The threat analysis and intended exposure stated in source comments have not been verified; review before production use | Security review required |
