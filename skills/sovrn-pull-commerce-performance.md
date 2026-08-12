---
name: Pull Sovrn Commerce performance data
description: >-
  Query Sovrn's real-time Commerce reports — transactions, merchants, links, pages,
  merchandise, networks and CUIDs — without tripping the one-request-per-minute limit.
api: openapi/sovrn-commerce-reports-openapi.yml
operations:
  - GET /reports/transactions
  - GET /reports/merchants
  - GET /reports/merchantsbydate
  - GET /reports/links
  - GET /reports/pages
  - GET /reports/merchandise
  - GET /reports/networks
  - GET /reports/cuids
---

# Pull Sovrn Commerce performance data

Eight endpoints on `https://viglink.io/v1` return the same commission data aggregated on a
different dimension. Pick the endpoint that matches the question; do not pull transactions
and aggregate client-side unless you actually need transaction-level detail.

## Before you start

Authenticate with the site's Commerce Secret Key: `Authorization: secret {SECRET_KEY}`.

**Budget your calls first.** These endpoints are limited to **1 request every 60 seconds**.
A dashboard that refreshes seven panels on load will fail six of them. Sequence the pulls
with at least a 60-second gap, or pull transactions once and derive the rest yourself.

## Choosing the endpoint

| Question | Endpoint |
|---|---|
| Every individual commission event | `GET /reports/transactions` |
| Which merchants earn me the most | `GET /reports/merchants` |
| Merchant performance day by day | `GET /reports/merchantsbydate` |
| Which links convert | `GET /reports/links` |
| Which pages convert | `GET /reports/pages` |
| Which products sell | `GET /reports/merchandise` |
| Which affiliate networks pay | `GET /reports/networks` |
| Performance by my own click identifier | `GET /reports/cuids` |

## Steps

1. **Bound the window with a date filter.** These endpoints publish **no pagination
   parameters** — volume is controlled entirely by dates. Use `clickDate`, or
   `clickDateStart` + `clickDateEnd`, or `commissionDate`, or `updateDate`. `updateDate` is
   the one to use for incremental sync, because commissions are revised after the fact and a
   click-date query will not show you a correction.
2. **Narrow further as needed** with `campaignIds`, `merchantGroupIds`, `country`,
   `deviceType`, `programType`, `sovrnProduct`, `cuids`, `subIds`, and the UTM filters
   (`linkUtmSource`/`Medium`/`Campaign`/`Term`/`Content` and the `pageUtm*` equivalents).
3. **Read the envelope.** Every aggregate endpoint returns `{data, totals}` where totals use
   the shared rollup shape: `revenue`, `clicks`, `sales`, `actions`, `conversionRate`, `epc`.
   `/reports/transactions` is the exception — it returns `{transactions: [...]}` of
   `TransactionModel`.
4. **Join on the real keys.** `TransactionModel` carries `revenueId` (the event),
   `clickSid`, `campaignId`, `merchantGroupId`, `cuid`, `linkUtmInfo`, `pageUtmInfo`,
   `destinationUrl`, `pageUrl` and `merchandise[]`. Note that the merchant key is
   `merchantGroupId` here but `groupId` in Merchant Summaries and `group_id` in Promo Codes —
   normalise before joining across products.
5. **Wait 60 seconds before the next call.**

## Rules

- **A 404 does not mean "broken".** These endpoints return 404 with phrasings like "Merchant
  data not found" for an empty result set as well as for an unknown resource. Treat 404 as
  "no rows" unless you have other evidence.
- **The 5XX response is a wildcard.** The spec declares `5XX: Unexpected error` rather than
  enumerated statuses. Retry with backoff.
- **No 429 is declared** on these endpoints even though the limit exists, and no
  `Retry-After` or `RateLimit-*` header is documented. You cannot detect exhaustion from
  headers — you have to pace by the clock.
- **Two duplicated UTM sets.** `linkUtmInfo` describes the link, `pageUtmInfo` the page they
  were on. They are not interchangeable.

## Related

- `rate-limits/sovrn-rate-limits.yml` — every published Sovrn limit in one place.
- `data-model/sovrn-data-model.yml` — the entity graph and the id-naming divergence.
- MCP equivalents: `trx_Transactions`, `trx_Merchants`, `trx_Merchants_By_Date`, `trx_Links`,
  `trx_Pages`, `trx_Merchandise`, `trx_Networks`, `trx_CUIDs` at
  <https://mcp.sovrn.com/commerce>.
