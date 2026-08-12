---
name: Price a click with Sovrn Commerce Bid Check
description: >-
  Get a real-time bid on a single click before routing traffic, and decide between sending
  the user through Sovrn or to a fallback destination.
api: openapi/sovrn-commerce-bid-check-openapi.yml
operations:
  - getBid
---

# Price a click with Sovrn Commerce Bid Check

Bid Check answers one question at request time: *is this specific click worth routing through
Sovrn right now?* It replaces the stale historical EPC that Link Check returns with a live
valuation that expires.

## Before you start

- You need the **site Commerce API key** for the site or traffic source where the click will
  originate. It is passed as the `key` query parameter.
- You need the **real end user's** IP address and User-Agent. Both are required, and both
  must be the end user's, not your server's — a bid priced against your data-centre IP is
  worthless.

## Steps

1. **Call `getBid`** — `GET https://api.viglink.com/api/bid` with the required parameters:
   - `key` — site Commerce API key
   - `out` — the destination URL, URL-encoded
   - `ip` — the end user's IP (IPv4 or IPv6)
   - `userAgent` — the end user's browser User-Agent string
2. **Add the optional levers you care about:**
   - `bidFloor` — minimum acceptable bid in USD as a decimal
   - `includeCpa=true` — consider CPA offers alongside CPC bids
   - `referrerUrl` — the page the click came from, URL-encoded
   - `subId` — fully qualified SubID URL to associate the click with a child campaign
   - `cuid` and the five `utm_*` parameters for attribution
3. **Pass consent through.** `gdprApplies`, `gdprConsent`, `ccpaConsent` and `gppConsent`
   accept raw IAB consent strings. If GDPR applies to the user and you have a TCF string,
   send it. When `gdprApplies` is omitted Sovrn determines applicability itself.
4. **Branch on the response.** The 200 body is one of two shapes:
   - **BidWin** — `affiliated: true`, plus `pricing` (`CPC` for a real-time bid win, `CPA`
     when an offer wins), `eepc` (expected earnings per click in USD, already net of Sovrn's
     revenue share), `url` (the Sovrn redirect URL to send the user to) and `expireInMs`.
   - **BidNoFill** — `affiliated: false` and nothing else. There is no bid.
5. **Respect `expireInMs`.** A bid is a perishable quote. If you cannot route the click
   within that window, the price you were quoted no longer applies. Do not cache the `url`
   beyond it and do not reuse a bid across users.
6. **Route or fall back.** On a win, redirect the user to `url`. On a no-fill — or when
   `eepc` is below what the placement is worth to you — send the user to the original
   destination directly.

## Rules

- **`eepc` means different things for CPC and CPA.** For a CPC bid it is the rate you earn by
  routing this click before the bid expires. For a CPA offer it is an *average* Sovrn
  expects, and is explicitly **not guaranteed for the individual click**. Do not present a
  CPA `eepc` as a committed payout.
- **One call per click.** Bid Check prices a specific click from a specific user; it is not a
  catalogue lookup and there is no batch form.
- **No documented rate limit** on Bid Check, and no `Retry-After` header anywhere in the
  Sovrn estate. Back off exponentially on errors rather than hammering.
- **`key` is a query parameter, not the Secret Key.** Bid Check declares no securityScheme in
  its OpenAPI at all; the credential is the site Commerce API key in the URL.

## Related

- `skills/sovrn-monetize-a-destination-url.md` — historical EPC and simple link wrapping.
- `rate-limits/sovrn-rate-limits.yml`
- Sovrn's launch note: <https://www.sovrn.com/blog/sovrn-launches-commerce-bid-check/>
