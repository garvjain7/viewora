# API conventions

Applies to the public HTTP API (dashboard, storefront, third parties later). Source of truth is the generated OpenAPI; this file states the rules the generator must respect.

## 1. URLs and versioning
- Base: `https://api.viewora.com/v1`. Version in the path; breaking change = new major, old one kept for a stated period.
- Plural nouns, kebab-case path segments, lowercase: `/v1/products/{product_id}/variants`.
- Public storefront pages are not API: `viewora.com/{merchant-slug}/{product-slug}` and `viewora.com/q/{code}`.
- Reserved top-level slugs (cannot be merchant slugs): `q`, `api`, `app`, `admin`, `static`, `assets`, `help`, `login`, `signup`, `search`, `cart`, `checkout`, `orders`, `legal`, `about`.
- IDs in paths are UUIDv7 strings. Never expose sequential integers (`product_seq` is internal).

## 2. Methods and status codes
| Action | Method | Success |
|---|---|---|
| Read one / list | GET | 200 |
| Create | POST | 201 + `Location` |
| Replace / partial update | PUT / PATCH (JSON Merge Patch) | 200 |
| Delete | DELETE | 204 |
| Long-running work | POST | 202 + job resource |

Errors: 400 malformed, 401 unauthenticated, 403 forbidden, 404 not found (also for cross-tenant ids, never 403, to avoid leaking existence), 409 conflict, 412 precondition failed, 422 validation, 429 rate limited (with `Retry-After`), 5xx server. No 200-with-error bodies.

## 3. Request and response
- JSON, UTF-8, `snake_case` fields.
- Timestamps ISO 8601 UTC with `Z`. Money as `{"amount": 129900, "currency": "INR"}` where `amount` is integer paise. No floats.
- Enums are lowercase strings. Unknown enum values from clients are rejected, not ignored.
- Responses never contain internal fields (`tenant_id` is implicit, `product_seq` is hidden).
- Every response carries `X-Request-Id`; clients may send one.

## 4. Error format
One shape everywhere (RFC 9457 problem details):
```json
{
  "type": "https://docs.viewora.com/errors/out-of-stock",
  "title": "Out of stock",
  "status": 409,
  "code": "inventory.out_of_stock",
  "detail": "Variant 0192… has 0 units available.",
  "request_id": "01JC…",
  "errors": [{"field": "items[0].quantity", "code": "max", "message": "…"}]
}
```
`code` is stable and machine-readable (`module.reason`); `detail` is human text and may change.

## 5. Pagination, filtering, sorting
- Cursor pagination only: `?limit=50&cursor=…`; `limit` max 100, default 25. Response: `{"items":[…], "next_cursor": "…"|null}`.
- Cursors are opaque, signed, and encode the sort key + id. No `OFFSET`.
- Filters are explicit query params (`?status=published&category_id=…`). Sorting: `?sort=-created_at` with an allow-list per endpoint.

## 6. Idempotency
- `POST` endpoints that create orders, payments, refunds, subscriptions, uploads or credits **require** `Idempotency-Key` (client-generated UUID). (COR-03)
- Stored 24 h in State Redis with Postgres unique constraint as the backstop; same key + same body returns the original response; same key + different body returns 422 `idempotency.key_reuse`.

## 7. Concurrency
- Mutable resources return an `ETag`; updates may send `If-Match`. Mismatch → 412.

## 8. Auth
- `Authorization: Bearer <access token>`; 15-minute lifetime. Refresh via `/v1/auth/refresh` with a rotating refresh token; reuse detection revokes the family.
- Guest shoppers use a signed anonymous session cookie for cart only.
- Webhooks: `POST /v1/webhooks/razorpay` verified by signature; no bearer token; replay-safe by event id.

## 9. Rate limiting
- Token bucket in State Redis, per IP and per tenant, per route class (auth, read, write, expensive). Headers: `RateLimit-Limit`, `RateLimit-Remaining`, `RateLimit-Reset`. 429 on exhaustion. Expensive routes (AI, try-on, generation) also check credits and return 402 `billing.insufficient_credits`.

## 10. Caching
- Public GETs set `Cache-Control` explicitly with `s-maxage` and `stale-while-revalidate`, and `Vary` only on what truly varies.
- Authenticated and tenant-scoped responses are `private, no-store` unless stated.
- Asset URLs are content-hash paths with `immutable, max-age=31536000`.

## 11. Async operations
- Long work returns 202 with `{"job_id": "…", "status_url": "/v1/jobs/{id}"}`. Job states: `queued, running, succeeded, failed, cancelled`. Result links, not payloads.
- Uploads: `POST /v1/assets/uploads` returns a signed URL + `asset_id`; the client uploads directly, then `POST /v1/assets/{id}/complete`.

## 12. Events and webhooks out
- Outgoing event payloads have `id`, `type`, `created_at`, `data`; types are `module.noun.verb` (`orders.order.paid`). Delivery is at-least-once; receivers dedupe by `id`.

## 13. Deprecation and change policy
- Additive changes (new optional fields, new endpoints) are non-breaking. Removing/renaming fields, changing types or semantics is breaking.
- Deprecations are announced with `Deprecation` and `Sunset` headers and kept for a documented period.
- CI diffs the OpenAPI against the last release and fails on breaking changes without a version bump.

## 14. Naming of error codes and metrics
- Error codes: `identity.invalid_otp`, `billing.limit_reached`, `inventory.out_of_stock`, `payments.signature_invalid`.
- Metrics: `viewora_http_request_duration_seconds{route,method,status}`, `viewora_stream_lag{stream}`, `viewora_cache_hit_ratio{ns}`.   