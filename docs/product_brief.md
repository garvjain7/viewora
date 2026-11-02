## 00 — Product brief

## One line

Viewora lets a customer scan a QR code on a physical product or menu and instantly see it in 3D and AR, try it on where it makes sense, and buy it, all inside Viewora. Merchants pay a subscription.

## Problem

- Merchants selling visually-driven products (furniture, jewelry, glasses, clothing, footwear, gadgets, food) lose sales because customers cannot judge the product from flat photos.
- Existing 3D/AR tools are viewers bolted onto someone else's store. They are slow on mid-range phones and give the merchant no commerce, analytics or discovery.
- Marketplaces give commerce but no premium interactive experience and no QR-first entry from the physical world.

## Product

- **QR-first entry.** A printed QR resolves in <50 ms to a product page. Codes are immutable; their target can change.
- **Premium interactive page.** 3D viewer, AR placement, try-on (glasses on-device; clothing through a hosted provider; jewelry earrings/necklaces later).
- **Commerce inside Viewora.** Cart, checkout, orders, payments (Razorpay). Not a redirect to the merchant's site.
- **Discovery.** On scan, show similar products from across Viewora's inventory (cross-merchant), plus search, reviews and ratings.
- **AI layer.** Comparison, recommendations, suggestions, and analysis of merchant product pages with concrete fixes.
- **Merchant tools.** Catalog, inventory, QR management, analytics, plan/billing, 3D asset supply.
- **Later.** Connection to a last-mile delivery platform (separate product by the same owner).

## Users

| Who                                                        | Wants                                                               |
| ---------------------------------------------------------- | ------------------------------------------------------------------- |
| Shopper (scans in store or at home)                        | Fast page, believable 3D/AR, trust (reviews), simple checkout       |
| Merchant (shop, cafe, jeweller, optician, furniture store) | Product pages without a tech team, orders, evidence it drives sales |
| Viewora admin (the owner)                                  | Moderation, support, billing visibility, platform health            |

## Business model

- Subscription (SaaS), India only, INR. Plans gate catalog size, 3D asset supply, AI credits and try-on usage.
- **3D supply tiers:** merchant uploads own model (base tier); Viewora generates with AI (paid credits, QA gate); assisted digitization for hard categories such as glasses and jewelry.
- Free tier shape (permanent vs trial) is **OPEN** — see 05, A-01.

## Verticals

Restaurants and cafes, furniture, jewelry, footwear, fashion/clothing, glasses, electronics. Per-vertical differences live in *vertical profiles* (attribute schemas + viewer/try-on settings), not in forks of code. See [06-glossary](06-glossary.md).

## Differentiators (what we protect)

1. Speed on a mid-range Android phone over 4G. Budgets are in [01-requirements](01-requirements.md) (PERF-*).
2. QR-to-3D in one scan with no app install.
3. Try-on and AR that feel premium without heavy downloads.
4. Cross-merchant similar-products on scan.

## Non-goals

- Not a general marketplace clone with every Amazon feature. Features outside the modules in 01 need a requirement first.
- No native mobile app. Web only.
- No international markets, multi-currency or multi-language at launch.
- No own logistics (the delivery platform is later and separate).
- Tax/compliance depth is deferred, not forgotten; tracked in 05.
- No combined multi-merchant cart in v1 (ADR-0016, Proposed).

## Success measures

- Scan-to-3D-interactive p75 < 3 s on target device class.
- Merchant onboarding to first live QR in one sitting, self-serve.
- Conversion lift versus the same merchant's flat-image page (measured with holdout, see ANL-05).
- Paying merchants; cost per merchant under plan price at all gates (CST-*).

## Constraints

Solo builder. Python + FastAPI + React base. Budget and cloud credits are **OPEN** (05, A-06). The build is also a vehicle for learning production-scale practice, so decisions are made on merit and recorded with revisit triggers.
