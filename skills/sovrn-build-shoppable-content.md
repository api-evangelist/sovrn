---
name: Build shoppable content with Sovrn product data
description: >-
  Turn a piece of content into a monetized shopping surface — recommended products, price
  comparisons across merchants, and verified promo codes — using Sovrn's three product APIs.
api: openapi/sovrn-product-recommendations-openapi.yml
operations:
  - get_product_recommendations
  - GET /sites/{site-api-key}/compare/prices/{market}/by/accuracy
  - GET /product
---

# Build shoppable content with Sovrn product data

Three separate Sovrn APIs answer three different product questions. They live on three
different hosts, take their API key in three different places, and describe a product with
three different field sets — so decide which one you need before you call anything.

| You want | Call | Host |
|---|---|---|
| Products relevant to this content | `get_product_recommendations` | `shopping-gallery.prd-commerce.sovrnservices.com` |
| The same product across merchants, cheapest first | Price Comparisons | `comparisons.sovrn.com` |
| A working promo code for this product | Promo Codes | `viglink.io/coupons` |

## Recommend products for a piece of content

1. **`POST https://shopping-gallery.prd-commerce.sovrnservices.com/ai-orchestration/products`**
   (`get_product_recommendations`) with your content plus optional filters such as price
   range or preferred merchants. The API key goes in the `apiKey` **query parameter**.
2. **Set `pageUrl` deliberately — it is the cache key.** Responses are cached on the exact
   `pageUrl` value, so:
   - use a stable slug or the real page URL (`mens_shoes`, `/products/mens/shoes`)
   - one piece of content per `pageUrl`
   - give variant intents their own keys (`mens_shoes_budget` vs `mens_shoes_premium`)
   - **never put prompt or page text in `pageUrl`** — that belongs in `content`
   Getting this wrong means every page on your site shares one cached answer.
3. **Read the array back**: `id`, `name`, `imageURL`, `thumbnailURL`, `currency`,
   `salePrice`, `retailPrice`, `discountRate`, `inStock`, `affiliatable`, `deepLink`.
   Drop anything where `affiliatable` is false — you will earn nothing on it.
4. **Mind the two limits**: 30,000 requests per 5 minutes per key, but only **100 requests
   per 5 minutes per IP**. If your servers egress through one NAT or load balancer, the IP
   limit is what you will hit, not the key limit.

## Compare prices across merchants

5. **`GET https://comparisons.sovrn.com/api/affiliate/v3.5/sites/{site-api-key}/compare/prices/{market}/by/accuracy`**
   — the site API key is a **path segment** here, and `market` selects the geography.
6. You get an array of `Product`: `id`, `name`, `merchant` (`id`, `name`, `logo`),
   `deeplink`, `image`, `thumbnail`, `currency`, `salePrice`, `retailPrice`, `discountRate`,
   `affiliatable`, `epc`. Note `deeplink`/`image`/`thumbnail` here versus
   `deepLink`/`imageURL`/`thumbnailURL` in recommendations — same concept, different casing.
7. This endpoint is the fastest of the three: **100 requests per second across the whole
   account**. Sovrn recommends one set of keys unless you need per-site request counts.
8. If you would rather not call it server-side, the same data renders client-side through the
   Automated Comparisons library — see `components/sovrn-components.yml`.

## Attach a promo code

9. **`GET https://viglink.io/coupons/product`** with `product_url`, `api_key` (query
   parameter), optional `cuid`, the five `utm_*` parameters, and `include_unverified` if you
   want unverified codes.
10. The response carries `merchant` (`domain`, `group_id`, `group_name`, `logo_url`), `scan`
    (`verification_active`, `when_to_check_back`) and `coupons[]` — each with `code`,
    `affiliated_url`, `original_price`, `price_with_code`, `currency`, `verified`,
    `verified_at`, `code_description`.
11. **Show `verified` state honestly.** A code with `verified: false` may not work; if
    `scan.verification_active` is true, `when_to_check_back` tells you when a fresh
    verification will have run.

## Rules

- **Three key placements, one credential mental model to break.** `apiKey` (query) for
  recommendations, `{site-api-key}` (path) for comparisons, `api_key` (query) for promo
  codes, and `Authorization: secret …` (header) for everything else in Commerce.
- **`comparisons.sovrn.com` sits behind a WAF** that 403s unauthenticated probing. Expect to
  need a valid key even to explore.
- **Always carry `cuid` and UTM parameters through** to the affiliated URL, or the resulting
  commission cannot be attributed back to the piece of content in `/reports/pages` or
  `/reports/cuids`.

## Related

- `skills/sovrn-pull-commerce-performance.md` — reading the revenue back out.
- `components/sovrn-components.yml` — the client-side rendering path.
- MCP equivalents: `rec_recommend_products` and `comp_search_prices`.
