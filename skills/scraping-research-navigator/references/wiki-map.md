# scraping-wiki map

Snapshot of structure as of the 2026-09-23 wiki commit. Page counts grow daily, so treat `index.md` in the synced checkout as authoritative and use this file to know *where to look*.

## Layout

- `index.md` — the catalog. One line per page: `- [Title](path.md) — one-sentence summary`, grouped under headings. Grep it first; it is faster than opening pages.
- `concepts/` (~36) — techniques and ideas.
- `entities/` (~190) — one page per tool, library, browser, anti-bot vendor or proxy provider. Index groups them under *Anti-bot systems*, *Browsers*, *Proxy networks and tools*, *Tools and libraries*, *Web Unblockers*.
- `comparisons/` (3) — all about stealth/anti-detect tooling, so `[diagnose]`.
- `timelines/` (1) — Cloudflare bypass evolution, `[diagnose]`.
- `log.md` — the maintenance log; check it to see what changed recently.

Links between pages are relative markdown links (`[api-scraping](./api-scraping.md)`, `(../entities/camoufox.md)`), and most pages end with `## Related` and `## Sources` sections. Sources point back to TWSC newsletter articles on substack.

## Concepts by usefulness to a research crawler

### Build — apply directly

- `api-scraping.md` — finding and replaying internal JSON/GraphQL endpoints. Often the cleanest way to get retail prices with stable product IDs.
- `http-performance.md` — concurrency, connection reuse, bandwidth reduction; relevant to daily 10M+-row collection.
- `scraping-infrastructure.md` — scheduling, storage, monitoring at scale.
- `inventory-tracking.md` — stock/availability signals; relevant to the "product unavailable at an endpoint" problem.
- `location-data-scraping.md` — spatially indexed data (store locators, listings), relevant to city-level prices and rent.
- `ai-scraping-assistants.md` — using LLMs to write and repair scrapers when layouts change. This matches how a coding agent uses this skill: once, at development time, never inside the daily crawl.
- `webdriver-vs-cdp-vs-bidi.md` — choosing a browser-automation protocol.
- `websocket-scraping.md` — sites that push prices over WebSockets.
- `scraping-economics.md` — the cost structure of scraping.
- `web-scraping-legal-landscape.md` — CFAA, DMCA §1201, ToS, robots.txt, GDPR. US-centric; not legal advice.
- `agent-sandboxing.md` — safety for coding agents running scraper code.
- `proxy-fundamentals.md` — what proxies are. Fine for background; rotating proxies to dodge blocks is `[diagnose]`.

### Diagnose — understand, don't implement

Detection mechanics (useful for explaining a block): `bot-detection.md`, `ml-bot-detection.md`, `browser-fingerprinting.md`, `canvas-fingerprinting.md`, `tls-fingerprinting.md`, `cdp-detection.md`, `websocket-bot-detection.md`, `vm-based-javascript-obfuscation.md`, `ai-web-api-fingerprinting.md`, `browser-privacy-fingerprinting-defenses.md`.

Circumvention techniques: `anti-detect-browsers.md`, `hybrid-scraping.md` (browser for clearance, HTTP client afterwards), `cookie-session-reuse.md`, `homepage-first-navigation.md`, `mouse-movement-emulation.md`, `webrtc-ip-leak.md`, `static-isp-proxies.md`, `mobile-proxy-farming.md`, `web-unblockers.md`, `javascript-proxy-tracing.md`, `prerender-stealth-request.md`, `certificate-pinning.md`.

## Entities worth knowing for this project

Open-source libraries/frameworks (`[build]`): `scrapy.md`, `playwright.md`, `puppeteer.md`, `cheerio.md`, `aiohttp.md`, `python-requests.md`, `crawl4ai.md`, `lightpanda.md`, `mitmproxy.md` (a local debugging proxy, e.g. to inspect your own crawler's requests).

Commercial services (`[paid]`, don't recommend): e.g. `scrapegraphai.md` (LLM library plus commercial API), `browserbase-chrome.md`, `lucidextractor.md`, `sensecollect.md`, `spidra.md`, `intuned-agent.md`. Most unblocker and proxy-network pages are commercial too, and already `[diagnose]`. A freemium tool is acceptable only if its free tier alone is enough; state the limits.

Anti-bot vendor pages (`[diagnose]`: read to recognise the block you hit): `cloudflare.md`, `akamai.md`, `datadome.md`, `perimeterx.md`, `kasada.md`, `aws-waf.md`, `recaptcha.md`.

Stealth browsers, unblockers, proxy networks (`[diagnose]`, don't adopt): e.g. `camoufox.md`, `nodriver.md`, `pydoll.md`, `botasaurus.md`, `kameleo.md`, `gologin.md`, `dolphin-anty.md`, `rayobrowse.md`, `oxylabs-unblocker.md`, `residential-proxy.md`, `bright-data.md`.

Note that `curl-cffi.md` is a normal HTTP client but is covered in the wiki mainly for TLS impersonation; using it as a fast async client is fine, using its `impersonate=` feature to disguise the crawler is `[diagnose]`.
