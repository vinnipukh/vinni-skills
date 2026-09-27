# Acquisition strategy: the escalation ladder

The ladder itself is in `SKILL.md`, along with the project's rules (what's out of scope at every step). This file has the details for each step. Record which step each source ended up on and why. Whatever step a source ends on, its URL list comes from the sitemap (see "Sitemaps" below).

Tier 2 is often cheaper than Tier 1 once found: one JSON request can replace dozens of HTML page loads, with no rendering and cleaner fields. Still do the sitemap step before it, since that costs only a few requests. If you notice an obvious listing API during Tier 1, note it and move to it right after the sitemap check.

## Tier 0 — Pre-flight

Check rule 5 in `SKILL.md` first (no government websites). Then make at most 3 requests, a second or two apart:

1. **robots.txt.** Parse it with `urllib.robotparser` or `protego`. Note whether the path is allowed for your User-Agent, any `Crawl-delay`, and any `Sitemap:` lines. If robots.txt disallows the path, stop and note it; look for an allowed public path (sitemap, another category URL) instead.
2. **The page itself, once.** Record the status code, the size, and whether `ETag` or `Last-Modified` headers are sent (they allow conditional requests on repeat days). Then look for:
   - **the visible price in the raw HTML.** Search for every spelling of it: Turkish `1.249,90`, English `1,249.90` and plain `1249.90`. If the price has no decimals, also try the bare integer. If none is there, the price is rendered client-side;
   - **JSON-LD** `<script type="application/ld+json">` blocks with `Product`, `Offer` or `AggregateOffer` (`price`, `lowPrice`, `priceCurrency`, `sku`);
   - **framework state:** `__NEXT_DATA__`, `window.__NUXT__`, `__INITIAL_STATE__` / `__PRELOADED_STATE__`, `__APOLLO_STATE__`;
   - **microdata** `itemprop="price"`, or OpenGraph `product:price:amount`.
3. **`/sitemap.xml`**, only if robots.txt listed no sitemap.

Next step: price or structured data present → Tier 1. Price absent from the raw HTML → the sitemap step, then Tier 2. No visible price to search for and no structured data → get a visible price and check again.

**A vendor being present is not the same as a challenge being served.** Many Turkish retailers sit behind Cloudflare or Akamai and serve plain requests normally. *Present* only means the vendor's header, cookie or script is there:
- Cloudflare: `cf-ray` or `server: cloudflare` header, `__cf_bm` cookie, `/cdn-cgi/challenge-platform/`;
- Akamai: `server: akamai` or `x-akamai-*` header, `_abck` or `bm_sz` cookie;
- DataDome: `x-datadome` header, `captcha-delivery.com`;
- PerimeterX/HUMAN: `_px*` cookies, `px-cdn`;
- Imperva: `x-iinfo` header, `incap_ses_` / `visid_incap_` cookies;
- Kasada: `x-kpsdk-*` headers;
- AWS WAF: `x-amzn-waf-*` header, `aws-waf-token`.

A challenge was *served* only when the status is 403, 429 or 503, or the body is a challenge page: `_cf_chl_opt`, `<title>Just a moment...</title>`, `geo.captcha-delivery.com/captcha`, `px-captcha`, `_Incapsula_Resource?`, or a title like "Access Denied", "Attention Required", "Robot Check" or "Security Check". Record both in the source card's `probe` block. One check from one IP is a narrow sample: CI runners' datacenter IPs and daily repetition can be treated differently.

## Sitemaps: the map of what to crawl

Start every source from its sitemap. It gives the full list of product and category URLs without crawling pagination, and its `<lastmod>` dates can make the daily crawl incremental.

**Find and map it.** Take the `Sitemap:` lines from robots.txt, or `/sitemap.xml`. Read it politely: one request at a time, honouring `Crawl-delay`.
- A sitemap index lists child sitemaps. Walk them and decompress `.gz` files. Stop at a sensible cap (about 200 files), and say so if you hit it.
- Split the URLs into product, category and other. Match the URL *path* or the child sitemap's name (`sitemap-products-3.xml`, `kategori-sitemap.xml`), never the hostname: a host like `productshop.com` would fool the match.
  - Common product patterns: `product`, `urun`, `/p/`, `-p-<digits>`, `/dp/`, `pdp`, `/item/`.
  - Common category patterns: `categor`, `kategori`, `/c/`, `-c-<digits>`, `/k/`, `collection`, `/liste`, `department`, `brand`, `marka`.
  - If most URLs land in "other", look at a few and adjust the patterns for that site.
- Rate `<lastmod>` for each type:
  - `absent`: no dates, so incremental crawling isn't possible;
  - `sparse`: fewer than half the URLs have a date, so treat it as unreliable;
  - `suspect`: more than 90% of the URLs share one timestamp, which is usually the sitemap's build time, not a product change;
  - `plausible`: dates vary per URL.

**Category pages or product pages?** Choose by the fields you need.
- For **price and name only**, use category/listing pages, or the JSON they load. One request returns dozens of products, so this is the lightest way to cover a whole assortment every day.
- For **details that only the product page has** (package size, unit price, barcode, brand, specs), take the product URLs from the sitemap. Those details rarely change, so fetch each product page once, store the details by product ID, and re-fetch only when its `<lastmod>` changes.
- A common split: daily prices from category pages, and product pages only for new or changed products.

**Incremental daily crawl.** Have the crawler keep a state file per source, `state/sitemap-<source_id>.json`. Each day it fetches the URLs that are new or whose `<lastmod>` moved, plus a sample of about 20 unchanged ones.
- **The state is a per-URL status map:** lastmod, type, source sitemap, `first_seen`, `last_seen`, and a status of `listed` or `missing`. Write it atomically, and commit it with the data so every run, local or CI, continues from the last one.
- **Make the unchanged sample reproducible.** Seed it with the run date and the sitemap URL, so a rerun of the same day checks the same products.
- **Make partial scans safe.** If the file cap is hit or a child sitemap fails, URLs from the unread files keep their previous entry instead of looking "new" next run. Log a warning and exit `3`.
- A URL that a re-read sitemap file no longer lists is marked `missing`.

Rules:
- **Run at the same time every day.** Many sites give `<lastmod>` only to the day, so a fixed schedule keeps each run's window consistent.
- **Don't trust `<lastmod>` until it's verified for prices.** For at least a week, re-fetch the sampled unchanged products and compare their prices with the last stored ones. If any price changed while `<lastmod>` didn't, the site doesn't bump `<lastmod>` on price changes. Crawl that source in full: category pages keep that cheap. Record the result in the source card's `sitemap.lastmod_check`. Keep sampling after that, because a site can change how it builds sitemaps.
- **Mark carried-forward prices.** When a product is skipped because `<lastmod>` didn't move, store yesterday's price flagged `carried_forward: true`, with the date it was last actually observed. The index can then separate observed prices from carried ones, and report how much of each day was carried.
- **A URL disappearing from the sitemap is not proof that the product was delisted.** Sitemaps can be partial or rotated; confirm with the category page before treating a product as gone.
- `<lastmod>` also moves for non-price edits (stock, description). That only costs an extra fetch, which is fine.

**Free library alternatives**, if a crawler is already built on one of them. The knowledge sources don't cover reading sitemaps, so these come from the libraries' own docs:
- **Scrapy `SitemapSpider`** (BSD). `sitemap_urls` accepts a robots.txt URL. `sitemap_follow` picks which child sitemaps to walk, and `sitemap_rules` routes product vs category URLs to different callbacks. `sitemap_filter(entries)` receives each entry's `lastmod`, so an incremental filter is a few lines. Use it with the politeness settings from `research-crawling-practices.md`. https://docs.scrapy.org/en/latest/topics/spiders.html#sitemapspider
- **advertools `sitemap_to_df`** (MIT). Returns a pandas DataFrame with parsed `lastmod`, and supports a robots.txt input and `If-None-Match`. It downloads with several threads by default, so set `max_workers` to the per-site concurrency the user chose. https://advertools.readthedocs.io/en/master/advertools.sitemaps.html
- **ultimate-sitemap-parser** (GPL-3.0; mind the license if your crawler's code uses another). Error-tolerant, and finds sitemaps that robots.txt doesn't list. https://github.com/GateNLP/ultimate-sitemap-parser

Whichever you use, keep the rules above: verify `<lastmod>` before skipping products, flag carried prices, and stay within the per-site concurrency the user chose.

## Tier 1 — Raw HTML

Plain `requests`/`httpx` (or Scrapy for many pages). Before writing CSS selectors, parse the embedded data the pre-flight found (JSON-LD, framework state, microdata; see Tier 0), because it is more stable than layout. JSON-LD is common on Turkish retailers for SEO.

Reduce load: take URLs from the sitemap rather than following pagination, and fetch only new or changed products once `<lastmod>` is verified (see "Sitemaps" above); send `If-None-Match` / `If-Modified-Since` on repeat days (wiki: `concepts/http-performance.md` → *HTTP Caching*); reuse one connection pool.

**Escalate when:** the price isn't in the raw HTML (rendered client-side), listings only load via JS/infinite scroll, or you've spent the ~30 minute time-box.

## Tier 2 — Internal API

"Internal" means the requests the site's own public web pages make, visible to anyone in DevTools. It doesn't mean a developer/partner API the retailer publishes separately. The page's own request is only useful because it is what the page renders, so the validation gate below applies with full force here: some endpoints return list prices, member prices or a different store's prices than the page shows.

Find it:
1. Chrome DevTools → Network → filter **Fetch/XHR** → reload the page and scroll/paginate once.
2. Use the Network panel's search (Ctrl/Cmd+F) for a price or product name you can see. The response containing it is your endpoint.
3. Right-click → *Copy as cURL*. Replay it from the terminal and confirm it works.

Minimise it:
- Remove headers and cookies one at a time until the request breaks; keep only what's required. Document the required set per source.
- Find the pagination and page-size parameters. Prefer category/listing endpoints (many products per call) over per-product calls. Keep page size within what the site's own frontend uses, or modestly above it.
- Check for per-store / per-city parameters — relevant for regional price variation.

Wiki: `concepts/api-scraping.md` (sections *Authentication Patterns*, *Finding Endpoints on Pages That Have Changed*, *Geo-targeted APIs*).

### When the API refuses the request: calling it the way the site does

Many endpoints reject bare requests because they expect context the site's own frontend always supplies. Supplying that same context is using the endpoint as designed, and is fine:

- Headers the frontend sends: `Accept: application/json`, `Referer`/`Origin` of the page, `X-Requested-With`, `Accept-Language: tr-TR`, store/region/channel headers.
- Ordinary session state from one normal page load at the start of the run: session cookie, CSRF/XSRF token, selected store or city cookie, a public key embedded in the page's JS (one every visitor receives, not tied to any account). Reuse it for that run instead of reloading the page per request — this is where the load saving comes from.

Where it stops being "calling it like the site does" and becomes circumvention (`[diagnose]` — don't do it):

- The refusal is an anti-bot challenge: Cloudflare/Akamai/DataDome/PerimeterX/Kasada interstitial, a 403 whose body is challenge JavaScript, a CAPTCHA, or a 429 that persists after backing off.
- Harvesting anti-bot clearance cookies (`cf_clearance`, `_abck`, `datadome`, `_px*`) from a browser to replay in an HTTP client (wiki: `concepts/cookie-session-reuse.md`, `concepts/hybrid-scraping.md`).
- TLS/HTTP2 fingerprint impersonation, fingerprint forging, rotating IPs or identities after being blocked.

If you hit the first case, check the sitemap step if you haven't yet, then try Tier 3 with a normal browser at a low rate once. If that is also challenged, stop and record a coverage gap.

## Concurrency: the user chooses it

Concurrency is how many requests (or browser pages) the crawler has open at the same time. It has two parts:
- **per site:** how many requests go to one retailer at once;
- **total:** how many run at once across all retailers. For Tier 3, that is also how many browser pages are open, and each page costs memory.

This skill doesn't prescribe a number. Before writing or changing a crawler, explain the trade-off to the user and ask them to enter both numbers:
- **Higher** finishes the daily run sooner, but puts more load on the retailer's servers, makes rate limits and blocks more likely, and uses more memory on the user's machine.
- **Lower** is politer and more reliable, but the run takes longer.

Help them decide by reporting what you know:
- the site's `Crawl-delay`, if robots.txt sets one;
- how many requests a full run needs: category pages vs product pages, and the incremental share;
- the time window the run has to fit in, such as the scheduler's job limit;
- whether the site has already answered with 429 or 503;
- how much memory the machine has, if Tier 3 is involved.

Remind them that every run comes from one IP, so parallel agents or overlapping runs add to the per-site number.

Put both numbers in the crawler's configuration, not in the code, and record them in the source card's `politeness` block. If a site starts answering 429 or 503, lower the per-site number, or ask the user to, and never go around the limit instead.

## Tier 3 — Browser automation

Use a standard browser: Playwright (preferred — async-native, lighter than Selenium) or Selenium with stock Chromium/Firefox. Not anti-detect or "stealth" builds (Camoufox, nodriver, undetected-chromedriver, Kameleo, etc.).

Keep it cheap:
- One browser process, with the number of pages the user chose (see "Concurrency" above). Keep the per-site limit separate from the total, and wait between navigations.
- Block images, media, fonts and third-party analytics via request routing. This typically cuts bandwidth and CPU by more than half.
- Capture the JSON instead of parsing the DOM: wait for the page's own product request, the one you found in DevTools, and read the data from its response. This gets Tier 2-quality data while the browser handles session setup normally. Don't wait for `networkidle`: it waits for every analytics beacon and makes each page slower and heavier.
- Headless is fine. Close contexts regularly to cap memory.

How to build it with Playwright (docs: https://playwright.dev/python/docs/network):
- Block images, media and fonts with `page.route("**/*", ...)`, aborting requests whose `resource_type` is in that set.
- Wrap `page.goto(url, wait_until="domcontentloaded")` in `page.expect_response(...)` with a predicate that matches the product request you found in DevTools, then read `await response.json()`.
- Limit each site with its own `asyncio.Semaphore`, sized to the user's per-site number, and start as many workers as the user's total. Read both numbers from the crawler's configuration.
- Open the context with `browser.new_context(locale="tr-TR")`, and wait a few seconds with jitter between navigations to the same site.
- Catch errors per URL, record them in the output, and move on to the next URL.

## User-Agent policy

- **No names, links or repository URLs** in any User-Agent, including in CI.
- **Sites that only serve browsers:** use a standard, current browser UA. This is common practice, and it keeps the crawl from depending on what a company chooses to show non-browser clients. Say so in the data README.
- **Tier 3** uses the browser's own UA.
- **Never rotate** UAs or identities to get around rate limits or blocks.

## Validation gate (before accepting any tier)

A tier is accepted only when the prices it returns match what the website shows.

1. Pick at least 5 products (8–10 is better), spread across categories and including one on promotion.
2. Read each price off the live page in a normal browser: by hand, or with an agent's browser tool that reads the rendered page. That is the `expected` column.
3. Run the scraper for the same products at about the same time. That is the `scraped` column.
4. Compare the two columns. First normalise both to a decimal with two places: strip currency signs and text, and treat the last `.` or `,` followed by one or two digits as the decimal separator, so `1.249,90 TL`, `₺1.249,90` and `1,249.90` all become `1249.90`. The result is `pass` only when every price matches exactly, `partial` when some do, `fail` when none do, and `insufficient` with fewer than 5 samples.
5. Write the result into the card's `validation` block: result, sampled, matched, reference method, date, and the mismatches.

On `partial` or `fail`, look at the mismatches before escalating. A wrong JSON field (list price vs sale price), a missing store/city parameter or a stale cache is usually the cause, not the tier. Re-run the gate whenever the endpoint or parser changes, and periodically (for example monthly) even when nothing seems to have changed.

## Source card

Every onboarded or repaired source gets a card: `assets/source-card.schema.json` defines it, and `assets/source-card.example.json` is a filled-in example. It records:
- the tier and the concrete reason for it, plus every tier tried and its outcome;
- the probe result, including which anti-bot vendor is present and whether a challenge was actually served;
- the sitemap: URL counts, crawl mode (incremental / full / category pages), the `<lastmod>` verdict and its price check;
- the schedule: local crawl time, and the share of prices carried forward;
- how to fetch: endpoint template, pagination, and required header and cookie names (never values);
- politeness settings, and the validation result;
- coverage, including known gaps, and cost per run;
- a self-critique (assumptions, skipped steps, what could go stale);
- provenance: the commit of each knowledge source (`git log -1 --format='%h %cs'` in each checkout), and the wiki pages consulted.

Check it against the schema with any JSON Schema validator, or by reading the schema. One rule the schema can't express: an `active` card must have a `pass` validation. The daily run manifest then only needs, per source, the card's `source_id`, requests made, bytes transferred, runtime, status counts and rows extracted. The methods and coverage tables are generated from the cards plus manifests.
