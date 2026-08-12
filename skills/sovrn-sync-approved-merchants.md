---
name: Sync your approved Sovrn merchants and rates
description: >-
  Load the full set of merchants a Sovrn Commerce account can affiliate with, including
  geo-specific rates and performance, then keep it current with the delta endpoint instead of
  refetching.
api: openapi/sovrn-merchant-summaries-openapi.yml
operations:
  - POST /summaries
  - GET /summaries/delta
---

# Sync your approved Sovrn merchants and rates

This is the only Sovrn surface with a real incremental-sync contract. Use it, rather than
re-pulling the whole merchant list on a schedule.

## Before you start

Authenticate with `Authorization: secret {SECRET_KEY}`. The limit here is **1 request every
10 seconds** — tighter-feeling than the reporting APIs but ten times more permissive.
Exceeding it can get the key temporarily blocked, not just throttled.

## Steps

### First run — full load

1. **`POST https://viglink.io/merchants/rates/summaries`** with a JSON body containing:
   - `filters` — an array of `{type, values}` objects. Filter types are `NAME`, `GROUP_ID`,
     `DOMAIN`, `GEO`, `PROGRAM_TYPE` and `CATEGORY`. Omit for everything.
   - `page` — starting at 1
   - `pageSize` — default 1000, **maximum 2500**
2. **Read the pagination fields off the response**: `page`, `perPage`, `totalItems`, and the
   merchants in `results`.
3. **Page through**, leaving **10 seconds between every request**, including between pages.
   `totalItems / pageSize` tells you how many calls that is — check it before you start,
   because 100,000 merchants at 2500 per page is 40 calls and nearly seven minutes.

### Every run after — delta

4. **`GET https://viglink.io/merchants/rates/summaries/delta?since={timestamp}`** returns
   only merchants updated since that point.
5. **Send `If-None-Match`** with the ETag from your last successful call. A **304 Not
   Modified** means nothing has changed and you should do nothing — this is the only
   conditional-request contract Sovrn publishes anywhere, and honouring it is free.
6. **Send `Accept-Encoding`** for a compressed response; these payloads are large.

## What you get back

Each `MerchantGroupSummaryResponse` carries `groupId`, `name`, `description`, `terms`,
`sovrnPreferred`, `logoImageUrl`, `category` and a `sovrn` block of performance:

- `CPA` — `geo`, `averageEpc`, `averageOrderValue`, `calculatedCommissionRate`, `rates[]`,
  `domains[]`
- `CPC` — `calculatedEpc`

Each rate in `rates[]` is a `MerchantRateDto`: `currentRate`, `rateFormat`, `action`,
`details`. **Rates are geo-specific** — the same merchant pays differently by country, so a
single headline rate per merchant is a lossy summary.

## Rules

- **`groupId` here is `merchantGroupId` in reporting and `group_id` in promo codes.** Same
  merchant, three key names. Normalise on ingest.
- **A 429 is declared on both operations** — the only two Sovrn operations that declare one —
  but there is still no `Retry-After` header. Back off exponentially.
- **404 is "User not found" on the POST and "Resource not found" on the delta**, and is also
  what an empty result looks like. Do not treat it as a fatal error without checking.
- **Do not filter client-side what you can filter server-side.** Every filter you push into
  `filters` is a page you do not have to fetch at 10 seconds apiece.

## Related

- `conventions/sovrn-conventions.yml` — the delta-sync and conditional-request notes.
- `data-model/sovrn-data-model.yml` — MerchantGroup, MerchantRate, NetworkSummary.
