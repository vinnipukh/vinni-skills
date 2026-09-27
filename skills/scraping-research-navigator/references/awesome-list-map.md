# awesome-web-scraping map

Snapshot as of the 2026-09-20 commit. List the `###` headings of a language file for the live list.

## Layout

- `README.md` — topic links plus a few non-library sections. **Skip** "Captcha Solving Services" and "Proxy Server Marketplaces" (the latter link to forums such as blackhatworld).
- One list per language: `python.md`, `javascript.md`, `golang.md`, `ruby.md`, `php.md`, `java.md`; `cli.md` for command-line tools.
- `manuals.md` — articles and books on fundamentals (HTML, HTTP, DNS, TCP, TLS, WebSocket, concurrency, text encoding, URL, XMLHttpRequest). Good for explaining a concept to someone new.
- `AGENTS.md`, `spec/docs/*` — rules for contributing to the list. Only relevant if the user wants to submit an entry.

Headings use `## Topic` and `### Topic : Subtopic`. Entries are `* [name](url) - description`. The list is breadth-first and not curated for maintenance status: check a repo's last commit and issue tracker before adopting a library. Some entries are years old (for example `reppy`, the listed robots.txt parser, is unmaintained; the stdlib `urllib.robotparser` or `protego` are safer choices).

## Python sections most relevant to a price-index crawler

| Need | Section in `python.md` |
|---|---|
| HTTP clients (sync/async) | `Network : General`, `Network : Asynchronous` |
| Crawl frameworks | `Web Scraping : Frameworks` (Scrapy, Crawl4AI, frontera, …) |
| Crawl management UIs | `Web Scraping : Tools` (ScrapydWeb, Gerapy) |
| HTML parsing | `HTML/XML Parsing : General` |
| Main-content / structured-data extraction | `Content Extraction`, `HTML/XML Parsing : Metadata` |
| Headless browsers | `Browser Automation : Drivers` / `Frameworks` |
| Price and currency strings | `Text Processing : Price and Currency` (price-parser) |
| Dates in listings | `Text Processing : Date and Time` |
| robots.txt | `Text Processing : robots.txt` |
| Turkish text normalisation (ı/İ, ş, ğ…) | `Text Processing : Transliteration`, `… : Character Encoding`, `… : Slugify` |
| Job queues for daily runs | `Job Queue`, `Concurrency` |
| URL canonicalisation (for stable product IDs) | `URL and Network Address : URL` |
| Output formats | `Data Serialization`, `Structured Formats : General` |

The language lists are almost all open-source libraries, which fits the project's free-only rule. If an entry turns out to be a hosted or paid service, tag it `[paid]`; don't recommend it unless its free tier alone is enough. `Web Scraping : Frameworks` and `Content Extraction` also list LLM-based extractors; skip those and use the deterministic ones.

The list has no sitemap *reader* (only a sitemap generator, `python-sitemap`). For reading sitemaps, follow "Sitemaps" in `acquisition-strategy.md`, which also lists free libraries that do it.

`[diagnose]` sections — don't adopt for this project: `Web Scraping : Bypass Protection` (Cloudflare bypass modules and CAPTCHA-solver clients), and proxy sections when the purpose is rotating around blocks.

JavaScript equivalents live in `javascript.md` under similarly named headings (`Web Scraping : Frameworks`, `HTML/XML Parsing : General`, `Browser Automation`, `Text Processing : Money`).
