---
name: Monetize a destination URL with Sovrn Commerce
description: >-
  Check whether an outbound merchant URL can be affiliated by Sovrn Commerce, get its
  expected earnings per click, and turn it into a monetized affiliate link with attribution
  attached.
api: openapi/sovrn-commerce-link-check-openapi.yml
operations:
  - campaigns
  - link
---

# Monetize a destination URL with Sovrn Commerce

Use this when you have a plain merchant product URL and you want the Sovrn-affiliated
version of it, plus a read on whether affiliating it is worth doing.

## Before you start

- You need a **Commerce Secret Key** for the site, from
  [platform.sovrn.com/commerce/settings](https://platform.sovrn.com/commerce/settings).
  Send it as `Authorization: secret {SECRET_KEY}` — the literal word `secret`, a space, then
  the key. Keys are **per site**, not per account.
- You also need the **campaign API key** (a different, public key) for link construction.
  Read it with `campaigns` rather than hard-coding it.
- The campaign must be **approved** before `link` will work. Sovrn approves campaigns only
  after affiliate links are installed and real clicks have flowed, so on a new account expect
  this call to fail until approval lands.

## Steps

1. **Find the campaign and its key.** Call `campaigns`
   (`GET https://rest.viglink.com/api/account/campaigns/{search}`) with `search` set to the
   campaign name or campaignId, or to `PRIMARY` for the default one. The response gives you
   `campaignId`, `apiKey`, `name` and `approvalStatus`. **Stop here if `approvalStatus` is
   not `Approved`** — every downstream call will 401 or return nothing useful.
2. **Check the URL.** Call `link` (`GET https://api.viglink.com/api/link/`) with:
   - `out` — the destination URL, required
   - `key` — the campaign apiKey from step 1, required
   - `format=json` (the docs say always use json)
   - optionally `geo` to test affiliation for a specific country, and `fbu` + `bf` to set a
     fallback URL and bid floor
3. **Read the answer.** The response carries `affiliatable` (can Sovrn monetize this at all),
   `eepc` (estimated earnings per click) and `optimized` (the ready-to-use monetized URL).
   `eepc` is an **estimate from other publishers' results**, not a promise about yours — do
   not present it as guaranteed revenue.
4. **Use `optimized` as the href.** If you need to build the link yourself instead — for a
   custom integration or to add your own tracking — construct it directly:
   `https://sovrn.co?key={CAMPAIGN_API_KEY}&u={URL_ENCODED_DESTINATION}`. `u` must be fully
   URL-encoded.
5. **Attach attribution while you have the chance.** Add `cuid` (your own click identifier,
   up to 2048 alphanumeric characters) and any of `utm_source`, `utm_medium`, `utm_campaign`,
   `utm_term`, `utm_content`. These are the *only* way the click comes back to you in
   reporting later — `/reports/cuids` and the `linkUtm*`/`pageUtm*` filters read exactly
   these values. A link built without them is unattributable after the fact.

## Rules

- **Two different keys, two different places.** The Secret Key goes in the `Authorization`
  header and never in a URL. The campaign API key goes in the `key` query parameter and is
  visible in the rendered page by design. Do not swap them.
- **Non-affiliated input only.** If you pass an already-affiliated URL as `out`, you are
  wrapping someone else's affiliate link.
- **401 means the credential regime is wrong.** The error body is
  `{"error":{"errors":[{"reason":"invalidCredentials",…}]},"code":401,…}`. Check the
  `secret ` prefix first — a missing prefix is the most common cause.
- **No idempotency key exists.** These are GETs, so retrying is safe by method, but there is
  no Idempotency-Key contract anywhere in the Sovrn estate.

## Related

- `skills/sovrn-price-a-click-with-bid-check.md` — when you want a live bid instead of an
  historical estimate.
- `conventions/sovrn-conventions.yml` — the attribution parameter set and where it resurfaces.
- `errors/sovrn-problem-types.yml` — the error envelope.
