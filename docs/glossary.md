# 06 — Glossary

One meaning per word. If code, UI or docs use a term differently, fix the usage, not this file (unless the meaning truly changed).

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
| Vertical | Market segment: restaurant/cafe, furniture, jewelry, footwear, fashion, glasses, electronics. |
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
| Stream | Redis Stream carrying high-volume telemetry events. |
| Cache Redis / State Redis | The two Redis instances: disposable cache vs durable state (ADR-0007). |
| Singleflight | Collapsing concurrent cache misses for one key into a single load. |
| Discovery item | A row in the public `discovery_items` projection used for cross-merchant similarity. |
| In-stock bitmap | Redis bitmap indexed by dense `product_seq` telling availability. |
| Similar products | Items from across Viewora resembling the scanned product, filtered by availability. |
| Holdout | Randomly excluded group used to measure a feature's real effect. |
| Gate | Set of conditions for a stage: **P** prototype, **L** launch, **S** scale. Not a date. |
| Trigger | A measurable condition that reopens a decision (found in each ADR). |
| ADR | Architecture Decision Record. Append-only. |
| Port | Interface that isolates a replaceable dependency (SearchIndex, TryOnProvider, Fulfillment). |
| Module | Unit of code ownership inside the monolith with a public interface. |
| Public interface | The only part of a module other modules may import. |
| RLS | Postgres row-level security. |
| RUM | Real-user monitoring from actual devices. |
| DLT | Indian SMS template/sender registration required before sending SMS. |
| Last-mile platform | Owner's separate delivery product, integrated later via the Fulfillment port. |