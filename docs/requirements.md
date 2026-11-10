# 01 — Requirements

> **Working requirements draft, not a final contract.** The MUST/SHOULD/MAY words and P/L/S gates preserve the source document's prioritization, but they still need owner review against the product, budget, and implementation sequence. Do not infer that every requirement is approved or must be implemented now. Current stack direction and unresolved questions are in [stack.md](stack.md) and [open-questions.md](open-questions.md).

Gates are proposed planning labels: **P** prototype, **L** launch, **S** scale. `P` does not itself define the first slice; choose that slice from the agreed user journey. See [README](../README.md).

Priority words: **MUST** / **SHOULD** / **MAY**.

---

## Part A — Functional requirements (by module)

Modules are proposed product/domain groupings. The current code layout is a scaffold and does not yet implement these boundaries.

### identity (IDN)

| ID     | Requirement                                                                                        | Gate |
| ------ | -------------------------------------------------------------------------------------------------- | ---- |
| IDN-01 | MUST support merchant sign-up with email + OTP/password; merchant = tenant.                        | P    |
| IDN-02 | MUST support shopper accounts via phone OTP (SMS) with guest checkout allowed.                     | L    |
| IDN-03 | MUST support merchant staff users with roles: owner, manager, staff.                               | L    |
| IDN-04 | MUST issue short-lived access tokens and revocable refresh tokens; token storage/revocation mechanism remains open. | L    |
| IDN-05 | MUST rate-limit auth endpoints per IP and per identifier.                                          | L    |
| IDN-06 | SHOULD support Google sign-in for merchants.                                                       | L    |

### billing (BIL)

| ID     | Requirement                                                                                   | Gate |
| ------ | --------------------------------------------------------------------------------------------- | ---- |
| BIL-01 | MUST define plans with limits (products, 3D assets, AI credits, try-on calls, QR codes).      | L    |
| BIL-02 | MUST charge subscriptions via Razorpay Subscriptions and handle signed webhooks idempotently. | L    |
| BIL-03 | MUST enforce plan limits in one place (`entitlements`) used by all modules.                 | L    |
| BIL-04 | MUST handle trial, upgrade, downgrade, failed payment grace, and cancellation.                | L    |
| BIL-05 | MUST meter AI generation and try-on usage as credits with a ledger.                           | L    |

### catalog (CAT)

| ID     | Requirement                                                                                            | Gate |
| ------ | ------------------------------------------------------------------------------------------------------ | ---- |
| CAT-01 | MUST let merchants create products with title, description, price (paise), category, images, variants. | P    |
| CAT-02 | MUST validate vertical-specific attributes against the vertical profile JSON Schema.                   | P    |
| CAT-03 | MUST map every product to a canonical category (Google product taxonomy).                              | L    |
| CAT-04 | MUST give each product a slug unique within its merchant; slug changes keep old slugs redirecting.     | P    |
| CAT-05 | MUST support bulk import from CSV with row-level error reporting.                                      | L    |
| CAT-06 | SHOULD support draft/published/archived states and scheduled publish.                                  | L    |
| CAT-07 | MAY ingest from external sources (Shopify, etc.). Scope **OPEN**; see [open questions](open-questions.md). | S |
| CAT-08 | MUST support a restaurant/cafe menu shape (sections, items, dietary tags) as a vertical profile.       | L    |

### inventory (INV)

| ID     | Requirement                                                                  | Gate |
| ------ | ---------------------------------------------------------------------------- | ---- |
| INV-01 | MUST track stock per variant.                                                | P    |
| INV-02 | MUST reserve stock at checkout start with expiry, and commit on payment.     | L    |
| INV-03 | MUST never oversell: reservation is a conditional atomic update in Postgres. | L    |
| INV-04 | MUST emit a stock-change event; cache overlays are derived from it.          | L    |
| INV-05 | MAY allow "unlimited" for menu items and made-to-order.                      | P    |

### assets (AST)

| ID     | Requirement                                                                                                                                                | Gate |
| ------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------- | ---- |
| AST-01 | MUST accept merchant GLB/glTF uploads via direct-to-storage signed URL (bytes never through the API).                                                      | P    |
| AST-02 | MUST validate and optimise models in a sandboxed worker: meshopt + KTX2, LOD0/LOD1, triangle and texture budgets, scale check against declared dimensions. | L    |
| AST-03 | MUST produce poster image, thumbnail and USDZ (iOS AR) for each model.                                                                                     | L    |
| AST-04 | MUST serve assets at immutable content-hash URLs via CDN.                                                                                                  | L    |
| AST-05 | MUST process 2D images to AVIF/WebP with srcset.                                                                                                           | L    |
| AST-06 | MUST reject or flag models that fail budgets, with an actionable message.                                                                                  | L    |
| AST-07 | SHOULD offer AI generation of a 3D model from photos (credits, QA gate before publish).                                                                    | L    |
| AST-08 | SHOULD offer an assisted-digitization request flow for glasses and jewelry.                                                                                | L    |
| AST-09 | MUST keep original uploads; derived assets are reproducible from them.                                                                                     | L    |

### qr (QRS)

| ID     | Requirement                                                                                                        | Gate |
| ------ | ------------------------------------------------------------------------------------------------------------------ | ---- |
| QRS-01 | MUST generate immutable codes at `viewora.com/q/{code}`; retargetable to any product/collection of the merchant. | P    |
| QRS-02 | MUST resolve a code in <50 ms p75 server time via cache, with a redirect (302) to the canonical product URL.       | L    |
| QRS-03 | MUST provide printable QR assets (SVG/PNG/PDF) with error-correction suitable for print.                           | L    |
| QRS-04 | MUST record each scan as an event (code, time, coarse device, no PII).                                             | L    |
| QRS-05 | MUST keep working codes if a merchant is suspended (show a neutral page, not a 404).                               | L    |
| QRS-06 | If legacy ScanVista QR codes are live, provide a redirect map. Whether this requirement applies is **OPEN**; see [open questions](open-questions.md). | L |

### storefront (STF)

| ID     | Requirement                                                                                  | Gate |
| ------ | -------------------------------------------------------------------------------------------- | ---- |
| STF-01 | MUST serve product pages at `viewora.com/{merchant-slug}/{product-slug}`, server-rendered. | P    |
| STF-02 | MUST show the 3D viewer with a poster placeholder and lazy-load the viewer runtime.          | P    |
| STF-03 | MUST offer AR launch (WebXR/Scene Viewer on Android, Quick Look on iOS).                     | L    |
| STF-04 | MUST degrade: no WebGL → poster + image gallery; slow network → LOD0 or poster only.       | L    |
| STF-05 | MUST provide merchant storefront home and collection pages.                                  | L    |
| STF-06 | MUST emit structured data (schema.org Product) and OG tags.                                  | L    |
| STF-07 | MUST be usable at 360 px width and keyboard accessible (WCAG 2.1 AA target).                 | L    |
| STF-08 | SHOULD support merchant branding (logo, colors, font from a fixed set).                      | L    |

### try-on (TRY)

| ID     | Requirement                                                                                                  | Gate |
| ------ | ------------------------------------------------------------------------------------------------------------ | ---- |
| TRY-01 | MUST provide on-device glasses try-on: face landmarks, smoothing, occlusion, scale from pupillary distance.  | L    |
| TRY-02 | MUST keep camera frames on device for on-device try-on; nothing uploaded.                                    | L    |
| TRY-03 | MUST provide clothing try-on through a hosted provider behind a `TryOnProvider` interface; async, metered. | L    |
| TRY-04 | MUST obtain explicit consent before photo upload and delete uploaded photos within 24 h.                     | L    |
| TRY-05 | MUST store per-product try-on metadata (anchor points, scale) and validate before enabling.                  | L    |
| TRY-06 | SHOULD provide size recommendation for clothing built in-house from merchant size charts and user inputs.    | L    |
| TRY-07 | MAY add earrings and necklaces on-device. Rings and footwear are out of scope for now.                       | S    |

### search (SRC)

| ID     | Requirement                                                                                                   | Gate |
| ------ | ------------------------------------------------------------------------------------------------------------- | ---- |
| SRC-01 | MUST provide keyword search with typo tolerance and filters (category, price, vertical attributes).           | L    |
| SRC-02 | MUST be backed by Postgres full-text + trigram at first; engine replacement is trigger-based (ADR-0004 area). | L    |
| SRC-03 | MUST provide autocomplete.                                                                                    | L    |
| SRC-04 | MUST scope search to a merchant on storefront and to all merchants on the Viewora-wide surface.               | L    |
| SRC-05 | SHOULD rank using text relevance, availability, rating and recency; weights in config, not code.              | L    |

### discovery (DSC) — similar products

| ID     | Requirement                                                                                                                                                           | Gate |
| ------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---- |
| DSC-01 | MUST show similar products on product page and after scan, from across all merchants who opted in.                                                                    | L    |
| DSC-02 | MUST compute candidates offline per category family using image/text embeddings and structured attributes (color, type); never scan the full catalog at request time. | L    |
| DSC-03 | MUST filter by availability at render using the in-stock bitmap.                                                                                                      | L    |
| DSC-04 | MUST rank same-merchant items first, then cross-merchant; at most 2 per merchant per list; collapse near-duplicates.                                                  | L    |
| DSC-05 | MUST read only from the public `discovery_items` projection (no tenant-private fields).                                                                             | L    |
| DSC-06 | MUST let merchants opt in/out; opt-in carries reciprocity (their items appear on others' pages only if they opt in).                                                  | L    |
| DSC-07 | MUST click out to the other merchant's Viewora storefront page; no combined cart in v1 (ADR-0016, Proposed).                                                          | L    |
| DSC-08 | MUST measure with a holdout (no-similar-products group).                                                                                                              | L    |

### cart (CRT)

| ID     | Requirement                                                                                                                                    | Gate |
| ------ | ---------------------------------------------------------------------------------------------------------------------------------------------- | ---- |
| CRT-01 | MUST keep a single-merchant cart for guests and accounts. Redis is the current storage candidate; define expiry, recovery/loss tolerance, and the exact transition into a durable order. Do not persist abandoned carts unless that requirement changes. | L    |
| CRT-02 | MUST re-price and re-check stock from Postgres at checkout; cached values are never trusted.                                                   | L    |
| CRT-03 | MUST merge guest cart into account cart on sign-in.                                                                                            | L    |

### orders (ORD) and payments (PAY)

| ID     | Requirement                                                                                                      | Gate |
| ------ | ---------------------------------------------------------------------------------------------------------------- | ---- |
| ORD-01 | MUST create orders from checkout with a persisted state machine (created, paid, fulfilled, cancelled, refunded). | L    |
| ORD-02 | MUST accept an `Idempotency-Key` on order creation.                                                            | L    |
| ORD-03 | MUST snapshot price, tax-inclusive total and address at order time.                                              | L    |
| ORD-04 | MUST provide order history for shoppers and an order list/status flow for merchants.                             | L    |
| PAY-01 | MUST use Razorpay hosted checkout; no card data touches Viewora.                                                 | L    |
| PAY-02 | MUST verify webhook signatures and process events idempotently and out of order.                                 | L    |
| PAY-03 | MUST reconcile Razorpay payments against orders daily and flag mismatches.                                       | L    |
| PAY-04 | MUST support refunds (full/partial) initiated by the merchant.                                                   | L    |
| PAY-05 | MAY support split settlement (Razorpay Route) to merchants. Decision **OPEN**.                             | S    |

### reviews (REV)

| ID     | Requirement                                                                     | Gate |
| ------ | ------------------------------------------------------------------------------- | ---- |
| REV-01 | MUST allow verified-purchase ratings and reviews with text and optional photos. | L    |
| REV-02 | MUST moderate reviews (queue, report, merchant reply).                          | L    |
| REV-03 | MUST compute rating aggregates asynchronously, served from the product overlay. | L    |
| REV-04 | SHOULD detect spam/duplicate reviews.                                           | S    |

### analytics (ANL)

| ID     | Requirement                                                                                                                  | Gate |
| ------ | ---------------------------------------------------------------------------------------------------------------------------- | ---- |
| ANL-01 | MUST record scan, view, 3D interaction, AR launch, try-on start, add-to-cart, and purchase events and deliver them to durable analytics storage. Redis Streams is a candidate transport, not a settled requirement. | L    |
| ANL-02 | MUST give merchants dashboards: scans, views, conversion, top products, device split.                                        | L    |
| ANL-03 | MUST aggregate in rollup tables; raw events retained for a bounded period.                                                   | L    |
| ANL-04 | MUST not collect PII in scan/view events.                                                                                    | L    |
| ANL-05 | MUST support holdout experiments (used by DSC-08 and AI suggestions).                                                        | L    |

### ai (AIX)

| ID     | Requirement                                                                                                                                 | Gate |
| ------ | ------------------------------------------------------------------------------------------------------------------------------------------- | ---- |
| AIX-01 | MUST provide product comparison (side by side, with a short generated summary) for 2–3 products.                                           | L    |
| AIX-02 | MUST provide personalised recommendations from session behavior, without requiring login.                                                   | S    |
| AIX-03 | MUST analyse a merchant's product page and return ranked, concrete suggestions (title, images, missing 3D, price outliers, attribute gaps). | L    |
| AIX-04 | MUST run all LLM/AI calls through one gateway with budget, timeout, retries, fallback, and logging.                                         | L    |
| AIX-05 | MUST NOT put an LLM call in the critical path of page render or checkout.                                                                   | L    |
| AIX-06 | SHOULD cache AI outputs by input hash.                                                                                                      | L    |
| AIX-07 | MUST help shoppers research products through natural-language questions, comparisons, and recommendations. Multi-turn behavior, personalization, and any purchase actions need explicit scope before implementation. | L |
| AIX-08 | MUST evaluate shopper-assistant quality on realistic multi-turn product tasks and equivalent requests written in varied styles; include typo robustness and factual/product constraint checks. | L |

### notifications (NTF), fulfillment (FUL), admin (ADM), integrations (INT)

| ID     | Requirement                                                                                               | Gate |
| ------ | --------------------------------------------------------------------------------------------------------- | ---- |
| NTF-01 | MUST send transactional email and SMS (DLT-registered templates) for OTP, order, payment, billing events. | L    |
| NTF-02 | MUST respect opt-outs and keep a send log.                                                                | L    |
| FUL-01 | MUST record shipping/pickup choice, status updates, and tracking reference per order.                     | L    |
| FUL-02 | MAY integrate the owner's last-mile delivery platform through a `Fulfillment` port.                     | S    |
| ADM-01 | MUST provide an admin console: merchants, suspension, moderation queue, asset QA queue, refunds view.     | L    |
| ADM-02 | MUST audit-log every admin and merchant-destructive action.                                               | L    |
| INT-01 | MAY expose a public API/embed (viewer widget on merchant's own site). Scope **OPEN**; see [open questions](open-questions.md). | S |

---

## Part B — Non-functional requirements

Targets are measured at **p75 on a mid-range Android phone over 4G** for client metrics and at the stated percentile server-side. Each has a gate and a way to verify.

### Performance (PERF)

| ID      | Requirement                         | Target                          | Gate | Verify                         |
| ------- | ----------------------------------- | ------------------------------- | ---- | ------------------------------ |
| PERF-01 | QR resolve server time              | p75 < 50 ms (cache hit < 10 ms) | L    | load test + RUM                |
| PERF-02 | Storefront TTFB                     | p75 < 200 ms                    | L    | RUM                            |
| PERF-03 | LCP on product page                 | ≤ 1.5 s                        | L    | RUM, Lighthouse CI             |
| PERF-04 | 3D interactive                      | < 3 s after navigation          | L    | RUM custom mark                |
| PERF-05 | First-view 3D payload               | ≤ 3 MB (LOD0, meshopt+KTX2)    | L    | asset budget check in pipeline |
| PERF-06 | Try-on (glasses)                    | ≥ 24 fps; start < 2 s          | L    | device lab                     |
| PERF-07 | API read p95                        | < 200 ms                        | L    | load test                      |
| PERF-08 | Create order p95                    | < 500 ms                        | L    | load test                      |
| PERF-09 | Search p95                          | < 150 ms                        | L    | load test                      |
| PERF-10 | Similar-products endpoint           | p95 < 80 ms cached              | L    | load test                      |
| PERF-11 | Storefront JS (initial, gz)         | ≤ 100 KB; viewer/try-on lazy   | L    | CI size budget                 |
| PERF-12 | Statement budget per endpoint       | stated per endpoint; no N+1     | P    | test asserts statement count   |
| PERF-13 | Cache hit ratio (product aggregate) | ≥ 95% at steady state          | L    | metrics                        |
| PERF-14 | Asset processing                    | model ready < 5 min p95         | L    | queue metrics                  |
| PERF-15 | CLS                                 | < 0.1                           | L    | RUM                            |

### Scalability (SCAL)

| ID      | Requirement                                                                                                     | Gate |
| ------- | --------------------------------------------------------------------------------------------------------------- | ---- |
| SCAL-01 | API and SSR stateless; scale horizontally behind a load balancer.                                               | L    |
| SCAL-02 | Workers scale independently by queue backlog; asset/generation workers scale to zero.                           | L    |
| SCAL-03 | Read path survives Postgres primary loss for hot content for ≥ 5 min via CDN + Redis (stale-while-revalidate). | L    |
| SCAL-04 | Scan/view event ingestion absorbs 10× bursts without dropping events beyond an explicitly agreed loss/retention policy. Redis Streams is one candidate transport. | L |
| SCAL-05 | Postgres read replica serves search/analytics reads; replica lag monitored.                                     | S    |
| SCAL-06 | Partition large tables (events, discovery) by key; documented keys.                                             | S    |
| SCAL-07 | Tenant isolation does not degrade with tenant count (RLS + indexed tenant_id).                                  | L    |
| SCAL-08 | A single noisy merchant cannot starve others (per-tenant rate and queue limits).                                | L    |

### Availability and recovery (AVL)

| ID     | Requirement                                                                                         | Gate |
| ------ | --------------------------------------------------------------------------------------------------- | ---- |
| AVL-01 | Public read path availability 99.9% monthly.                                                        | L    |
| AVL-02 | Checkout/payment path 99.5% monthly.                                                                | L    |
| AVL-03 | RPO ≤ 5 min, RTO ≤ 1 h; restore tested.                                                           | L    |
| AVL-04 | API/SSR run ≥ 2 tasks across 2 AZs.                                                                | L    |
| AVL-05 | Graceful degradation order: AI off → try-on off → similar off → 3D poster only → checkout last. | L    |

### Correctness (COR)

| ID     | Requirement                                                                 | Gate |
| ------ | --------------------------------------------------------------------------- | ---- |
| COR-01 | No oversell under concurrent checkout.                                      | L    |
| COR-02 | Money is integer paise; no floats.                                          | P    |
| COR-03 | All state-changing endpoints that create resources accept idempotency keys. | L    |
| COR-04 | Event consumers idempotent; at-least-once is assumed.                       | L    |
| COR-05 | A durable state change and the intent to perform required asynchronous work are recorded atomically or through a recoverable outbox. Exact queue integration depends on the queue decision. | L |
| COR-06 | Webhook handling tolerates replay and reorder.                              | L    |
| COR-07 | Test suite passes with all caches disabled.                                 | P    |

### Security (SEC)

| ID     | Requirement                                                                                            | Gate |
| ------ | ------------------------------------------------------------------------------------------------------ | ---- |
| SEC-01 | Tenant isolation via Postgres RLS, FORCE RLS, non-owner app role; cross-tenant tests in CI.            | P    |
| SEC-02 | Uploads untrusted: type sniffing, size limits, sandboxed processing (no network, CPU/mem/time limits). | L    |
| SEC-03 | Secrets in a secret manager; none in repo or images.                                                   | L    |
| SEC-04 | TLS everywhere; HSTS; strict CSP on storefront.                                                        | L    |
| SEC-05 | Passwords hashed with Argon2id; OTP rate-limited and single-use.                                       | L    |
| SEC-06 | Dependency and container scanning in CI; pinned lockfiles.                                             | L    |
| SEC-07 | Admin actions require 2FA.                                                                             | L    |
| SEC-08 | Signed URLs for private assets; originals not public.                                                  | L    |
| SEC-09 | Pen-test style review of auth, checkout, webhooks before launch.                                       | L    |

### Privacy (PRV)

| ID     | Requirement                                                                | Gate |
| ------ | -------------------------------------------------------------------------- | ---- |
| PRV-01 | Camera frames for on-device try-on never leave the device.                 | L    |
| PRV-02 | Uploaded try-on photos deleted ≤ 24 h; consent recorded.                  | L    |
| PRV-03 | Analytics events carry no PII; visitor id is rotating and non-identifying. | L    |
| PRV-04 | Data export and deletion for shopper accounts.                             | L    |
| PRV-05 | Logs scrub phone, email, address, tokens.                                  | L    |
| PRV-06 | Data sent to external AI providers is limited to the information needed for the request; provider data retention and use are reviewed before enabling a provider. No provider receives database credentials or unrestricted database access. | L |

### Compliance (CMP) — deferred

| ID     | Requirement                                                      | Gate         |
| ------ | ---------------------------------------------------------------- | ------------ |
| CMP-01 | India data-protection law (DPDP) obligations mapped to features. | L (deferred) |
| CMP-02 | GST invoicing and rates.                                         | L (deferred) |
| CMP-03 | E-commerce rules, consumer protection, grievance officer.        | L (deferred) |
| CMP-04 | DLT registration for SMS. Lead time makes this start early.      | L            |

### Observability (OBS)

| ID     | Requirement                                                                    | Gate |
| ------ | ------------------------------------------------------------------------------ | ---- |
| OBS-01 | Structured JSON logs with request id, tenant id, trace id.                     | P    |
| OBS-02 | OpenTelemetry traces across API, workers, outbox.                              | L    |
| OBS-03 | RED metrics per endpoint; queue depth, stream lag, cache hit ratio dashboards. | L    |
| OBS-04 | Real-user monitoring for the Web Vitals and 3D/try-on marks.                   | L    |
| OBS-05 | Alerts tied to SLOs with error-budget burn rates.                              | L    |

### Cost (CST)

| ID     | Requirement                                                          | Gate |
| ------ | -------------------------------------------------------------------- | ---- |
| CST-01 | Per-merchant unit cost known (storage, egress, compute, AI credits). | L    |
| CST-02 | Plan prices exceed worst-case unit cost with margin.                 | L    |
| CST-03 | Hard budget caps on AI generation and try-on provider spend.         | L    |

### Maintainability (MNT)

| ID     | Requirement                                                                          | Gate |
| ------ | ------------------------------------------------------------------------------------ | ---- |
| MNT-01 | Module boundaries enforced by lint; cross-module imports only via public interface.  | P    |
| MNT-02 | OpenAPI is generated from code; TS client generated from OpenAPI; CI fails on drift. | P    |
| MNT-03 | Migrations are expand/contract; CI regenerates `schema.sql` and fails on drift.    | P    |
| MNT-04 | One-command local environment.                                                       | P    |
| MNT-05 | CI < 10 min for the main pipeline.                                                   | L    |
| MNT-06 | Feature flags for risky features.                                                    | L    |
| MNT-07 | ADR for any new infrastructure component or >20 KB storefront dependency.            | P    |

### AI quality (AI)

| ID    | Requirement                                                                          | Gate |
| ----- | ------------------------------------------------------------------------------------ | ---- |
| AI-01 | Generated 3D models pass an automated QA score and a human gate before publish.      | L    |
| AI-02 | LLM outputs for merchants are labelled, reproducible by input hash, and non-binding. | L    |
| AI-03 | Offline eval set for similar-products relevance; regressions block model changes.    | L    |
| AI-04 | Provider swap possible without code changes outside the gateway.                     | L    |

---

## Part C — Traceability

Every FR belongs to exactly one module (the prefix). NFRs cross modules. ADRs cite the IDs they satisfy. When a requirement changes, edit it here and add an ADR only if a *decision* changed.
