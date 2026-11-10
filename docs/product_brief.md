## 00 — Product brief

> **Starting brief, not a frozen product specification.** This records the current concept and needs owner review as Viewora is shaped. See [working stack direction](stack.md) and [open questions](open-questions.md); neither is a final commitment.

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
- **AI layer.** Help shoppers research and decide between products with natural-language questions, comparisons, and recommendations; help merchants improve product pages with concrete suggestions. The assistant's multi-turn behavior, personalization, actions, and conversation retention remain open.
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
- Free tier shape (permanent vs trial) is **OPEN** — see [open questions](open-questions.md).

## AI research input (not an architecture template)

Independent Rufus research is useful as an evaluation prompt for Viewora's shopper-facing AI, not as evidence about Rufus's private architecture. The [FAccT 2025 chatbot audit](https://facctconference.org/static/docs/facct2025-206archivalpdfs/facct2025-final986-acmpaginated.pdf) reports response-quality differences for prompts written in different English dialects, worsened by typos. A [study of 40 people shopping with Rufus](https://arxiv.org/abs/2509.21501) examines multi-turn interaction traces and feedback. These motivate testing task completion, constraint accuracy, follow-up behavior, and varied prompt styles. They do not establish Viewora's model, data sources, schema, or system architecture.

## Verticals

Restaurants and cafes, furniture, jewelry, footwear, fashion/clothing, glasses, electronics are candidate verticals. Which vertical launches first is open. Per-vertical differences are intended to live in *vertical profiles* (attribute schemas + viewer/try-on settings), not forks of code. See [glossary](glossary.md).

## Differentiators (what we protect)

1. Speed on a mid-range Android phone over 4G. Candidate budgets are in [requirements.md](requirements.md) (PERF-*); they need validation against target devices and cost.
2. QR-to-3D in one scan with no app install.
3. Try-on and AR that feel premium without heavy downloads.
4. Cross-merchant similar-products on scan.

## Non-goals

- Not a general marketplace clone with every Amazon feature. Features outside the agreed product scope need a requirement first.
- No native mobile app. Web only.
- No international markets, multi-currency or multi-language at launch.
- No own logistics (the delivery platform is later and separate).
- Tax/compliance depth is not defined yet; see [open questions](open-questions.md).
- No combined multi-merchant cart in v1 (ADR-0016, Proposed).

## Success measures

- Scan-to-3D-interactive p75 < 3 s on target device class.
- Merchant onboarding to first live QR in one sitting, self-serve.
- Conversion lift versus the same merchant's flat-image page (measured with holdout, see ANL-05).
- Paying merchants; cost per merchant under plan price at all gates (CST-*).

## Constraints

Solo builder. Current stack direction is Python + FastAPI for the business API and React for web apps; the shopper storefront is planned as SSR (Node runtime/framework provisional) and the dashboard as a static SPA. See [stack.md](stack.md). Budget and cloud credits remain open. The project is intended to grow from a small working slice; this brief does not require all listed domains at the start.
