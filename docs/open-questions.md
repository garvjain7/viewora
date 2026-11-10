# Open questions and planning gaps

> **Status: working list, not decisions.** Items below are unresolved or need validation. The list is based on the current docs, scaffold, and owner discussions; it does not imply that every item must be settled before the first code slice.

## Product and AI

- What is the first merchant vertical and end-to-end user journey to implement?
- What is the free plan/trial shape, initial pricing, and launch country/compliance boundary? India/INR appears in the current brief, but commercial and legal scope still needs owner review.
- Should shoppers be able to create accounts at the first launch, or is guest checkout sufficient for the initial slice?
- Should cross-merchant discovery click out to another Viewora store, and is a combined cart excluded from the first release?
- Is the initial merchant catalog populated manually, by CSV, or by an external commerce integration? Shopify ingestion and a third-party viewer/widget API are later-scope candidates, not current implementations.
- Are there live ScanVista records or printed QR codes that must be supported? The source repository/deployment is not available in this workspace; do not assume the historical mapping table is accurate.
- The product brief describes shopper recommendations and comparison, while requirements focus more on comparison and merchant page analysis. Define the shopper AI assistant's intended tasks, multi-turn behavior, personalization, action boundaries, and success measures.
- Decide what conversational data, if any, must be retained for continuity, quality evaluation, safety, or analytics; define consent, minimization, retention, deletion, and provider-sharing rules before storing prompts or responses.
- Treat independent Rufus research as evaluation input, not evidence of Rufus's private architecture. Relevant external work includes [the FAccT 2025 dialect-based chatbot audit](https://facctconference.org/static/docs/facct2025-206archivalpdfs/facct2025-final986-acmpaginated.pdf) and [a human study of multi-turn shopping with Rufus](https://arxiv.org/abs/2509.21501). These support testing answer quality across prompt styles and real multi-turn tasks; they do not prescribe Viewora's schema or system design.
- MCP is an optional interoperability candidate for exposing or consuming AI capabilities. Clarify a concrete client, server, or integration use case before selecting an MCP SDK or deployment model.
- Select model providers and decide what catalog, shopper, and conversation data may be sent to each. Keep provider access behind Viewora services with application authorization; no direct model-to-database credentials.

## Stack and operations

- Confirm Neon plan support for PostgreSQL 15+, pgvector version/index operations, connection pooling, regions, backups, and the expected migration workflow.
- Confirm Render Redis-compatible service behavior for persistence, eviction policy, memory limits, Streams, failover, and recovery. The free tier is not assumed durable.
- Decide whether cache and state workloads need separate Redis instances, or whether the first slice can use logical separation with an accepted shared-failure risk.
- Decide which Redis uses are necessary for the first slice. The current proposal includes QR/product caching, rate limits, OTPs, cart state, idempotency coordination, telemetry streams, job progress, and an inventory bitmap; these are candidates, not a mandate to implement every use.
- Choose the background job system. Procrastinate is currently in Python dependencies and is PostgreSQL-backed; Celery/Arq with Redis are alternatives. Choose after comparing failure recovery, retry semantics, monitoring, and service capabilities.
- Resolve the mismatch between `identity.refresh_tokens` in the schema snapshot and older notes describing a Redis revocation list. Decide token format, rotation/reuse handling, durable record, and any Redis acceleration explicitly.
- Decide which telemetry must be durable and whether Redis Streams is a suitable first transport; define retention, backpressure, replay, and loss behavior.
- Confirm Node version, package manager/version, SSR framework, and where the Node storefront is deployed. The current React Router v7 SSR and Vite dashboard references are candidates, not implemented apps.
- Decide QR resolution boundary (Python API, edge function, or another handler) and cache invalidation when a QR target or product slug changes.
- Define cart expiry and acceptable data loss/recovery behavior. Checkout must reload price and stock from PostgreSQL regardless of Redis design.
- Define discovery bitmap rebuild/update semantics. It is a derived availability filter; checkout inventory remains authoritative in PostgreSQL.
- Choose migration source/workflow. `schema.sql` is intended as a generated full snapshot for review/AI context, but the current repo has no migration history or generator.
- Deployment host and production topology remain open. Docker is excluded from the initial local scaffold unless a concrete need is agreed.

## Product and schema

- Resolve whether any ScanVista data or printed QR codes need migration. The prototype source is not present in this workspace, so the current mapping table is unverified.
- Confirm the embedding model and vector dimension before creating production embeddings; the schema's `vector(768)` is explicitly provisional.
- Confirm category taxonomy source/seed and category-family mapping.
- Confirm whether promotions/coupons, returns/exchanges, product Q&A, merchant-side analytics, and reviews are needed in the first release or later. Their appearance/absence in the schema snapshot does not settle product scope.
- Clarify payments, returns, taxes/GST, promotions, reviews, product Q&A, account deletion, and compliance scope as their requirements enter implementation.
- Review the database snapshot against the intended first vertical and first release slice. Its breadth does not make all modeled domains final or immediately required.

