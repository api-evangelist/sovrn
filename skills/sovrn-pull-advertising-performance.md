---
name: Pull Sovrn Ad Exchange performance data
description: >-
  Query the Sovrn Advertising performance reporting API for web, CTV and mobile app ad
  performance, respecting its granularity windows and its separate credential regime.
api: openapi/sovrn-advertising-reporting-openapi.yml
operations:
  - GET /reporting/advertising/publishers/{publisherId}/
---

# Pull Sovrn Ad Exchange performance data

This is a **different product line from Commerce**, with a different host, a different
credential and no shared object model. Nothing you learned about the Commerce APIs transfers
here except the general shape of an HTTP call.

## Before you start

- Create an API key at
  [platform.sovrn.com/account/api-keys](https://platform.sovrn.com/account/api-keys). **Only
  the account owner** can create or delete keys, and **the value is shown once** — if you
  lose it you must delete the key and make a new one.
- Keys look like `{7-character prefix}.{32-character UUID}`.
- Send it as **`x-api-key: {KEY}`** — *not* the `Authorization: secret …` header the Commerce
  APIs take. Every key grants full access to all Advertising reporting data on the account;
  there is no scoping.
- You also need your **Publisher ID**.

## Steps

1. **Call `GET https://api.sovrn.com/reporting/advertising/publishers/{publisherId}/`** with
   all five required query parameters — the request fails without any one of them:
   - `start` — ISO 8601 UTC datetime, **inclusive** (`2024-08-01T00:00:00Z`)
   - `end` — ISO 8601 UTC datetime, **exclusive**
   - `metrics` — comma-separated (`impressions,publisherRevenue`)
   - `dimensions` — comma-separated (`country,demandPartner`)
   - `granularity` — `hour`, `day` or `month`
2. **Always use UTC with the trailing `Z`.** Timezone selection is not supported.
3. **Stay inside the window for your granularity** — the request is rejected otherwise:

   | Granularity | History available | Per-request window |
   |---|---|---|
   | `hour` | past 45 days | 1–24 hours |
   | `day` | past 2 years | 1 day to 1 month |
   | `month` | since January 2017 | 1 month to 1 year |

   Most dimensions only exist from 2022 onward, so a 2017 query returns a thin slice even
   where the month granularity technically reaches back.
4. **Filter with the optional parameters** as needed: `domain`, `bundleId`, `propertyType`,
   `auction`.
5. **Handle 204 as a real answer.** The API returns **204 No Content** when the query is
   valid but matches no data. That is not an error and not an outage.

## Rules

- **Some metrics and dimensions are temporarily unavailable** and Sovrn has published no
  restoration date: `requests`, `requestsWithBid`, `fill rate`, `zoneId`, `zoneSize`,
  `zoneName`. Requesting them will not give you data — check the reference before building a
  dashboard column on one.
- **403 is not 401.** 401 means the key is missing or malformed; 403 means the key is valid
  but not entitled to that publisher's data. Do not retry a 403 with the same key.
- **No request-rate limit is published** for this API — the constraints are the per-request
  data windows above, not a call rate.
- **Loop over windows, not over pages.** There is no pagination; a month of hourly data is 30
  or 31 separate day-shaped requests.

## Verify your key works

```
curl --request GET \
  --url 'https://api.sovrn.com/reporting/advertising/publishers/{PUBLISHER_ID}/?start=2024-08-01T00%3A00%3A00Z&end=2024-09-02T00%3A00%3A00Z&metrics=publisherRevenue&dimensions=domain&granularity=day' \
  --header 'x-api-key: {YOUR_KEY}'
```

## Related

- `authentication/sovrn-authentication.yml` — why this API's credential is not the Commerce one.
- Sovrn's own guide: <https://knowledge.sovrn.com/kb/how-do-i-use-the-sovrn-reporting-api>
