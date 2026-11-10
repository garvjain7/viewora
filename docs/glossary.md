# 06 — Glossary

> **Working glossary, not final product terminology.** These are current meanings used in planning notes. Revise them when product language or implementation changes.

| Term | Meaning |
|---|---|
| Merchant | A business paying for Viewora. Equals one **tenant**. |
| Tenant | Isolation unit in the database. Every tenant row has `tenant_id`; RLS enforces it. |
| Shopper | End customer who scans, browses and buys. May be a guest. |
| Staff | A user acting for a merchant (owner, manager, staff roles). |
| Admin | Viewora operator with the admin console. |
| Merchant slug | URL-safe unique merchant name in `viewora.com/{merchant-slug}/…`. |
| Product slug | URL-safe name unique within a merchant. Old slugs redirect. |
| Product | Sellable thing with title, category, attributes, media. Has one or more **variants**. |
| Variant | Concrete purchasable option (size/color/etc.) with SKU, price and stock. |
| Vertical | Candidate market segment: restaurant/cafe, furniture, jewelry, footwear, fashion, glasses, electronics. First launch vertical is open. |
| Vertical profile | Per-vertical config: attribute JSON Schema, viewer settings, try-on support, required assets. |
| Category | Node in the canonical taxonomy (Google product taxonomy). |
| Category family | Group of categories sharing one retrieval partition for similar-products. |
| QR code | Printed code pointing at `viewora.com/q/{code}`. The code string is immutable. |
| Target | What a QR code currently resolves to (product or collection). Changeable. |
| Scan | One resolve of a QR code. Recorded as an event, no PII. |
| Asset | Any stored file: original upload, derived model, poster, USDZ, image. |
| Original | The merchant-supplied upload, kept unmodified. |
| Derived asset | Output of the pipeline. Reproducible from the original. Lives at a content-hash URL. |
| LOD0 / LOD1 | Higher / lower detail model variants. LOD0 is the first-view model. |
| Poster | Static render of the 3D model, shown before the viewer loads. |
| Budget | Hard limit for triangles, texture size, payload bytes, or JS size. |
| Try-on | Seeing a product on the shopper's body. **On-device** (glasses) or **hosted** (clothing). |
| TryOnProvider | Interface to hosted virtual try-on vendors. |
| 3D supply tier | How a model is obtained: merchant upload, AI generation, assisted digitization. |
| Credit | Unit of metered usage (AI generation, hosted try-on) tracked in the ledger. |
| Entitlement | A plan limit resolved for a merchant, answered by billing only. |
| Overlay | Small fast-changing data (price, stock, rating) cached separately from the product aggregate. |
| Aggregate | The rarely-changing cached product representation (title, media, attributes). |
| Outbox | Table written in the same transaction as a state change; relayed to consumers. |
| Stream | Ordered event log abstraction. Redis Streams is a candidate for high-volume telemetry; transport, retention, and durability are open. |
| Cache role / state role | Logical Redis workload roles: rebuildable cache versus short-lived application state/coordination. They may or may not run on separate physical services; persistence and loss tolerance depend on the workload. |
| Singleflight | Collapsing concurrent cache misses for one key into a single load. |
| Discovery item | A row in the public `discovery_items` projection used for cross-merchant similarity. |
| In-stock bitmap | Candidate derived availability filter indexed by `product_seq`; may be stale and is not authoritative inventory. Checkout checks PostgreSQL. |
| Similar products | Items from across Viewora resembling the scanned product, filtered by availability. |
| Holdout | Randomly excluded group used to measure a feature's real effect. |
| Gate | Proposed planning stage: **P** prototype, **L** launch, **S** scale. Not a date or proof that requirements are approved. |
| Trigger | A measurable condition that may prompt a design review. |
| ADR | Architecture Decision Record. The source docs reference ADR IDs, but ADR files are not currently present in this workspace. |
| Port | Interface that isolates a replaceable dependency (SearchIndex, TryOnProvider, Fulfillment). |
| Module | Unit of code ownership inside the monolith with a public interface. |
| Public interface | The only part of a module other modules may import. |
| RLS | Postgres row-level security. |
| RUM | Real-user monitoring from actual devices. |
| DLT | Indian SMS template/sender registration required before sending SMS. |
| Last-mile platform | Owner's separate delivery product, integrated later via the Fulfillment port. |
