# Geo targeted proxies: which providers reach city, state or ASN level, what happens when no IP matches, and where ScraperAPI's country-only targeting fits (with a full plan table)

Two things decide whether a geo targeted proxy does the job you bought it for: the finest location level it can actually hit, and what the provider does when no IP exists at that level. Everything else, including the marketing page, is noise.

ScraperAPI sits at the simple end of that spectrum. Its geotargeting is controlled by a single `country_code` parameter, and the documentation goes no finer than countries. The `us` and `eu` codes work from the Hobby plan upward; the other country codes in its list, from Argentina and Australia to Japan, Kenya and South Korea, require Business or higher. If your work needs city, ZIP, ASN, geohash or coordinate level targeting, ScraperAPI will not do it and no plan upgrade changes that. If you need country-level exit IPs wrapped around a scraping API that handles rendering, retries and bot protection, the picture is different.

## Quick pick before the details

> **Most readers land on Hobby or Business.** Hobby if every request only needs a US or EU exit IP. Business the moment you need any other country, because Startup does not add a single country code to what Hobby already gives you. Both are billed monthly or annually, and ScraperAPI backs purchases with a 7-day no-questions-asked refund. 👉 [Check what Hobby and Business cost right now](https://www.scraperapi.com/pricing?fp_ref=coupons)

## Every ScraperAPI plan and the country codes it unlocks

Credit allowances and the monthly or annual price differ per tier and sit on the pricing page. What each tier changes about geotargeting and about running out of credits is below.

| Plan | Country codes available | Overflow and billing mechanics | Get started |
| --- | --- | --- | --- |
| Free | Not listed in the geotargeting plan table, which begins at Hobby | 1,000 trial credits, max 5 concurrent connections | [Try the free credits](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Hobby | `us`, `eu` only | Optional auto-upgrade to the next tier; scraping pauses at zero credits | [Choose Hobby](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Startup | `us`, `eu` only | Auto-upgrade available; scraping pauses at zero credits | [Choose Startup](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Business | Full country list | Auto-upgrade available | [Choose Business](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Scaling | Full country list | Auto-upgrade or pay-as-you-go up to a monthly cap you set | [Choose Scaling](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Professional | Full country list | Auto-upgrade or pay-as-you-go | [Choose Professional](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Advanced | Full country list | Pay-as-you-go at a fixed per-credit rate, capped monthly | [Choose Advanced](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Enterprise | Full country list, custom pricing | Requests to Ultra Premium carry their own minimum rate | [Talk to sales about Enterprise](https://www.scraperapi.com/pricing?fp_ref=coupons) |

Two details in that table matter more than the rest. Startup does not widen geotargeting at all, so if your reason for upgrading is a German or Japanese exit IP, the useful jump is Hobby to Business. And going over your allowance is not billed: requests beyond the limit are turned away with a 429 rather than queued, which is why pay-as-you-go exists as a separate opt-in on the higher tiers.

## The seven levels hiding behind the phrase "geo targeted proxies"

Country is the entry ticket. Providers that go further usually implement some subset of these, in roughly this order of precision:

**Country** uses ISO 3166-1 alpha-2 codes, lowercase in most APIs.

**State or region** follows ISO 3166-2. Bright Data accepts the standard two-letter codes for US states plus the 3-letter and numeric codes the standard defines for other regions, and requires lowercase.

**City** is where providers start to diverge. Bright Data states plainly that city targeting works best with residential proxies and that city targeting for its datacenter and ISP proxies has been deprecated entirely. So "we support city targeting" and "we support it on the pool you're buying" are separate questions.

**ZIP code** is residential-only at Bright Data. Oxylabs limits its `postalcode` parameter to 5-digit US ZIPs and requires the `cc-US` parameter alongside it.

**ASN** pins traffic to a specific ISP network. Oxylabs warns that the ASN parameter is overridden when `cc` is present in the same username string, and returns a 502 when no proxy is live on that network.

**Geohash** uses grid cells instead of administrative borders. Oxylabs documents four levels: 1 character covers roughly 5,000 km by 5,000 km, 2 characters roughly 1,250 by 625 km, 3 characters about 156 by 156 km (its example is Paris), 4 characters about 39 by 19.5 km.

**Coordinates** are the finest option and usually travel in a header rather than the username. Oxylabs takes a `Latitude:Longitude;Radius` value with a 10-mile minimum radius.

ScraperAPI stops after level one. That is a legitimate design choice for an API product: you pass `country_code=de` and the service handles proxy selection, rendering and anti-bot work inside the same request. It is the wrong product if your test requires knowing the exit IP is in Chicago rather than somewhere in Illinois.

State and city availability also depends on the country, not just the plan. Geonode's own documentation notes that city pools in some markets are thin and suggests using "Any" for better coverage. A provider advertising 195 locations is advertising the top of the funnel, not guaranteed inventory in every city on the list.

## Syntax differs per provider, and that is a real migration cost

Location flags are not portable. Each provider invented its own string format:

| Provider | How you set the location | Finest documented level |
| --- | --- | --- |
| ScraperAPI | `country_code` API parameter | Country |
| Bright Data | `-country-us` in the username, or `brd-exit-country` / `brd-exit-city` / `brd-exit-state` / `brd-exit-asn` / `brd-exit-zipcode` headers | ZIP and ASN, residential pools |
| Oxylabs | `cc`, `city`, `st`, `postalcode`, `geohash`, `ASN` inside the username; `X-Oxylabs-Geolocation` header for coordinates | Coordinates within a radius |
| Shifter | `country`, `region`, `city`, `asn`, `strict` flags appended with dashes | ASN, pool permitting |
| Geonode | `-country-`, `-state-`, `-city-`, `-asn-` appended after the username | ASN |

Multi-word place names are the usual trap. Bright Data wants `-city-sanfrancisco`, no spaces. Oxylabs wants underscores, as in `new_york`. Shifter wants lowercase slugs such as `region-new_south_wales`. Copying a working username from one provider to another will fail, and the failure often looks like an authentication error rather than a syntax error, which wastes an afternoon.

ScraperAPI avoids this class of problem by not offering the flags at all. Whether that reads as simplicity or as a missing feature depends entirely on your target list. 👉 [Open ScraperAPI and run one request with a country code before you commit](https://www.scraperapi.com/?fp_ref=coupons)

## When no IP exists at your target, three different things can happen

This is the part most comparison articles skip, and it decides whether a batch job finishes or dies halfway.

**Fail loudly.** Bright Data's shared pools let you set default countries at the zone level. If a default country is configured and no peer can be allocated there, the request fails and returns an error rather than silently routing elsewhere. That is good for data integrity and bad for unattended jobs.

**Fall back quietly, unless you opt out.** Shifter's default behavior is to fall back to a broader pool when nothing matches the full filter, so the request still completes. Add `strict-true` and you get an HTTP 502 instead. Its documentation frames the tradeoff directly: use strict matching when an exact geo match matters more than the request succeeding.

**Widen the grid step by step.** Oxylabs geohash targeting is strict by default and returns 502 when the cell is empty. The `geohash_fallback` option strips one character at a time, moving from a 4-character cell to 3, then 2, then 1, and only fails if even the single-character level has nothing.

None of these is wrong, but they are incompatible assumptions to build a pipeline on. If your code treats any 502 as "retry later", a provider that falls back silently will hand you data from the wrong country and you may never notice. If your code assumes the request always succeeds, a strict provider will produce gaps. Test one deliberately impossible target, such as a small city at 3 a.m. local time, and see which behavior you get.

For ScraperAPI, the country routing is documented, and the choice effectively disappears: there is one targeting level, so there is no narrower pool to fall back from.

## How to verify the exit IP before you trust a run

Every provider's documentation recommends the same sanity check: send a request through the proxy to an IP information endpoint and read back the city, region, ISP, latitude, longitude, timezone and postal code. Geonode's docs use `ip-api.com/json` for exactly this, and Oxylabs uses its own location endpoint for the same purpose.

Do this once per country code at the start of a project, not once ever. Proxy pools shift, and a code that resolved correctly in March can drift. If the response shows the right country but the wrong city, you have learned something important about which pool is serving you. ScraperAPI users can check the same way inside a scrape response.

## What a geotargeted request really costs in credits

ScraperAPI bills in credits, and one credit is the price of a standard page request. The multiplier is where budgets break:

- Standard pages: 1 credit
- Amazon: 5 credits
- Google and Bing, including subdomains: 25 credits
- LinkedIn: 30 credits
- Any site behind Cloudflare, DataDome or PerimeterX: an extra 10 credits when the protection is bypassed

Geotargeting is most often used for exactly the expensive categories. Local search results mean Google or Bing at 25 credits each, so the 1,000 free credits cover 40 SERP requests, not 1,000. A 100,000-credit allowance buys 4,000 localized searches if nothing else is running. Price your plan against the multiplier, not against the headline credit number, and use the Domain Cost Estimator in the dashboard plus the `max_cost` parameter to stop a single URL from quietly eating a large slice of the allowance.

Annual billing is where most of the discount lives, and ScraperAPI lets annual plans renew early for a fresh batch of credits, which matters if you burn through an allowance in month nine. Before you commit a year, confirm your target countries sit in the tier you're buying and that your main domains are priced at the multiplier you assumed.

## Picking a plan by job

**US and EU exits only, standard e-commerce or news pages.** Hobby. Nothing above it adds a country code you need, and Startup in particular adds none.

**Any other country on the list.** Business is the first tier that unlocks the full set. If your target list is one country in Europe plus the US, Hobby is enough; the moment it includes Brazil, India or Japan, you are buying Business.

**Localized Google or Bing results at volume.** Work out credits per request first (25 for each SERP call), then pick the allowance, and put Scaling or Professional on pay-as-you-go so a spike does not stop the run. Auto-upgrade is the cheaper-looking option that can move you up a tier mid-month, so decide which behavior you want before the invoice arrives.

**City, state, ZIP, ASN, geohash or coordinate precision.** ScraperAPI cannot serve this at any price. Bright Data, Oxylabs, Shifter and Geonode all document some subset of those levels, but check the pool: Bright Data's fine-grained targeting is residential-only, and Shifter's Country Geo and Non-Geo pools drop region, city and ASN filters entirely. Buying a residential plan and discovering your pool excludes city filters is the most common wasted purchase in this category.

**Sites that fight back.** The 10-credit surcharge for bypassing Cloudflare, DataDome or PerimeterX is worth paying if the alternative is maintaining your own headless browser fleet, and it is included with every plan rather than sold as an add-on. 👉 [Compare the plans that include premium proxies and JS rendering](https://www.scraperapi.com/pricing?fp_ref=coupons)

Two things to confirm before you pay. First, that every country on your target list is included in the tier you selected, since `us` and `eu` alone are a short list. Second, that your real cost per request accounts for domain multipliers, because a plan sized on standard pages will run dry fast on localized search. The 7-day refund gives you room to test both with real targets.

If country-level exit IPs plus a managed scraping layer is what you need, 👉 [start on Hobby and move to Business only when a country code forces it](https://www.scraperapi.com/?fp_ref=coupons). If your test cases name a city or an ASN, save the money and pick a residential provider instead.
