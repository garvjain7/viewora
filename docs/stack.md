# Working stack direction

> **Status: planning note, not a final architecture or contract.** This file records the current direction discussed for Viewora so code and other docs have a shared reference. Any choice may change as product needs, cost, and measured behavior become clearer. “Working direction” means the current starting point, not a permanent decision.

## Current direction

| Area | Working direction | Status and limits |
|---|---|---|
| Backend language | Python 3.13 | Current working direction. `backend/.python-version` and `backend/pyproject.toml` specify 3.13. |
| Backend framework | FastAPI | Present in backend dependencies; the API is not implemented yet. |
| Storefront | React, server rendered; React Router v7 is the current candidate | Provisional. If SSR runs on Node, Node serves rendered pages and calls the Python API. It is not a second business API by default. |
| Merchant dashboard | React SPA with Vite as the current candidate | Provisional. A built static dashboard can be served by a CDN; Node is needed for its build, not necessarily its production serving. |
| Database | PostgreSQL, version 15 or newer | Current minimum direction. Confirm the selected host's PostgreSQL and extension support before relying on vector search. |
| PostgreSQL provider | Neon | Current provider direction, still revisable. Development and production requirements, region, connection pooling, backups, and pgvector support need verification against the chosen plan. |
| Product asset storage | Cloudflare R2 with Cloudflare CDN | Current direction for photos, original and processed GLB/glTF, USDZ, posters, and other derivatives. The API should issue authorized upload details; large file bytes should go directly between client and object storage where feasible. |
| Redis-compatible service | Render Key Value / Redis-compatible service | Current provider direction, subject to plan capabilities. The free service is described in the README as temporary; do not assign it durable-state duties without confirming persistence, eviction, recovery, and service limits. |
| Redis responsibilities | Candidate uses: rebuildable cache, short-lived coordination/state, rate limiting, carts, telemetry streams, and discovery availability filters | These are separate workload roles, not a decision to run one or two physical Redis instances. Choose physical separation and persistence from failure and cost requirements. PostgreSQL remains authoritative for commerce, inventory, payments, and durable records. |
| Background jobs | Not selected | The Python dependencies currently include Procrastinate, which uses PostgreSQL. Redis-backed Celery/Arq or another queue is an alternative, not an established choice. Compare durability, retries, operations, and host constraints before choosing. |
| AI providers | External model/provider integrations are anticipated behind Viewora-owned application services | Provider choices and data-sharing terms are open. An external LLM must not receive database credentials or unrestricted SQL access. Viewora should authorize and execute narrowly scoped operations, then send only necessary context to the provider. |
| MCP | Optional integration candidate | MCP can standardize more than tool invocation, including resources, prompts, transports, and authorization flows. It is not required for Viewora's internal AI design; adopt it only for a concrete client/server interoperability need. |
| Database migrations and snapshot | Migration-managed database with a generated root `schema.sql` snapshot | The owner wants the complete snapshot kept easy for AI agents to inspect and never hand-edited. Alembic is present in dependencies, but the migration tool/workflow is not settled or implemented; current migration history is empty. |
| Local containers | No Docker Compose in the initial local scaffold | Explicit current instruction. Revisit only for an agreed local or deployment need. |

## Runtime boundaries

- **Python API:** owns business rules, authentication and tenant authorization, database transactions, provider adapters, and API responses.
- **Node storefront SSR (if retained):** renders shopper pages and calls the Python API for page data. Keep core commerce rules and direct database access out of this layer unless a later design explicitly needs otherwise.
- **Browser apps/packages:** dashboard UI, storefront hydration, `<model-viewer>` wrapper in `packages/viewer`, and Three.js/MediaPipe try-on in `packages/tryon`. Heavy viewer and try-on code should remain lazy-loaded.
- **Python workers:** perform background work using the selected queue mechanism. Durable job outcome and asset state need a recoverable record; transient live progress may use a separate channel if needed.
- **PostgreSQL:** durable source of truth for catalog, inventory, orders, payments, billing, and other persistent domain state.
- **Redis:** acceleration and coordination where loss/rebuild behavior is explicitly defined. It is not an authority for price, payment completion, or checkout inventory.
- **R2/CDN:** store and deliver asset bytes; PostgreSQL stores metadata and object keys.

## Status vocabulary

- **Working direction:** preferred starting assumption; may change.
- **Provisional:** selected for now but depends on a condition, evaluation, or measured trigger.
- **Open:** no choice made; do not implement as if settled.
- **Deferred:** deliberately outside the current implementation slice; revisit when requirements bring it into scope.
- **Implemented:** present in code and verified. A documented plan alone does not qualify.

