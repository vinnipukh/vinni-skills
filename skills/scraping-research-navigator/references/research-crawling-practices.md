# Research-grade crawling practices

How to turn what the sources say into crawler code that anyone can run for free on their own machine or in CI, and that anyone (including reviewers and other research groups) can check and reproduce from the published data.

## Politeness (write it into the code)

- Check `robots.txt` before crawling a host and cache the parsed result per run; `urllib.robotparser` or `protego` (Scrapy's parser) both work. Honour `Crawl-delay` when present.
- Follow the User-Agent policy in `acquisition-strategy.md`.
- Use the per-site and total concurrency the user chose (see "Concurrency" in `acquisition-strategy.md`), with a delay of a few seconds plus jitter between requests to the same site. In Scrapy, set `CONCURRENT_REQUESTS_PER_DOMAIN` and `CONCURRENT_REQUESTS` from the user's numbers, and use `DOWNLOAD_DELAY` and `AUTOTHROTTLE_ENABLED` for the delay.
- Take URLs from the sitemap instead of following pagination, as described in "Sitemaps" in `acquisition-strategy.md`.
- Back off on 429/503 (respect `Retry-After`), and stop crawling a host for the day after repeated 403s or challenge pages. Log it as a coverage gap rather than retrying with a different identity.

## Resumable runs

A crash halfway through should not mean fetching everything again; that doubles the load on retailers.

- **Dated, idempotent outputs.** Write each run to `data/raw/<source_id>/<YYYY-MM-DD>/…` and `data/manifests/<YYYY-MM-DD>.json`. Rerunning the same date overwrites or skips, never duplicates.
- **Checkpoint per source.** Keep `state/<YYYY-MM-DD>/<source_id>.json` with the last completed page/offset and the IDs already stored. `--resume` loads it and continues from there; without `--resume` the run starts clean. Sitemap status maps live beside them (`state/sitemap-<source_id>.json`). Commit `state/` with the data, so any machine or CI run continues where the last one stopped.
- **Atomic writes.** Write to `file.tmp`, then `os.replace()` it into place, so an interrupted run never leaves a half-written CSV or checkpoint.
- **Handle SIGTERM/Ctrl+C.** Flush the current checkpoint and manifest, then exit. Cron, Airflow and Actions all send SIGTERM on timeout or cancel.
- **Per-source isolation.** One source failing must not stop the others; record the failure in the manifest and move on.

## Running on a scheduler (cron, Airflow, GitHub Actions)

The daily crawl is ordinary scripts, never an LLM agent. Every entry point should follow this contract so any scheduler can run it unattended:

- **Non-interactive.** No prompts, no browser windows that wait for input. Configuration comes from CLI flags and environment variables (data/output paths), not from editing code.
- **Explicit date.** Take `--date YYYY-MM-DD` (default: today in `Europe/Istanbul`), so backfills and reruns are the same command.
- **Meaningful exit codes.** `0` done; `3` finished but some sources were blocked (a coverage gap, recorded in the manifest, not a job failure); anything else is a real error.
- **Record the actual time.** Store the fetch timestamp on every row; the scheduled time is a target, not a guarantee.
- **Logs to stdout/stderr**, one line per source summary, so the scheduler's log view is enough to debug.
- **No overlapping runs.** Two crawls at once would double the load on every site. Use `flock` with cron, `max_active_runs=1` in Airflow, and a `concurrency` group in Actions.
- **Bounded runtime.** Set timeouts per request and per source, and keep the whole run well under the scheduler's limit.

**GitHub Actions** (the default for publishing openly). Write the workflow from the official docs: https://docs.github.com/en/actions (overview) and https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax (syntax). It needs:
- an `on: schedule` cron trigger, plus `workflow_dispatch` for manual reruns;
- a `concurrency` group with `cancel-in-progress: false`, so two crawls never overlap;
- `permissions: contents: write`, so the job can commit;
- `TZ: Europe/Istanbul` in the job's environment, so `--date` matches the Turkish day;
- a crawl step that treats exit code `3` as success (for example `... || [ $? -eq 3 ]`);
- a final step with `if: always()` that commits `data/` and `state/` back to the repository, so the next run starts from them, even after a partial run;
- a `timeout-minutes` well below the job limit.

Know the platform's limits:
- Schedules run in UTC and usually start within a few minutes to half an hour of the set time.
- A scheduled workflow is disabled after 60 days without commits in a public repo; the daily data commit prevents that.
- Runners use shared datacenter IPs that change every run. Some retailers block them; record that as a coverage gap for CI runs rather than adding proxies.
- A job is killed after 6 hours.
- The repository grows every day. Commit extracted JSON/CSV, compress raw responses, and move old raw files to a release asset if the repo gets large.

**cron** (your own machine):

```cron
# crontab -e   (07:15 every day; flock skips the run if yesterday's is still going)
15 7 * * * cd /path/to/crawler && flock -n logs/daily-crawl.lock python3 -m crawler.daily --date "$(date +\%F)" --resume >> logs/cron.log 2>&1
```

**Airflow**, if you already run it. Follow the official documentation rather than a template from here, because the APIs change between major versions:
- Tutorial, "Airflow 101": https://airflow.apache.org/docs/apache-airflow/stable/tutorial/fundamentals.html
- BashOperator (Airflow 3 import path `airflow.providers.standard.operators.bash`; a non-zero exit fails the task, `skip_on_exit_code` changes that): https://airflow.apache.org/docs/apache-airflow-providers-standard/stable/operators/bash.html
- Template variables: https://airflow.apache.org/docs/apache-airflow/stable/templates-ref.html

Two things to get right in Airflow:
- `{{ ds }}` is the logical date, the *start* of the data interval, so a daily run is labelled with the previous day. Pass `{{ data_interval_end | ds }}` as `--date`.
- Set `max_active_runs=1` so crawls never overlap.

## Reproducibility

- Store raw responses (or at least the extracted JSON) alongside the parsed CSV, keyed by retailer, URL, and fetch timestamp, so parsing bugs can be fixed retroactively.
- Write a per-run manifest: crawler git commit, start/end time, hosts attempted, pages fetched, HTTP status counts, rows extracted, and hosts skipped with reason. This makes the coverage tables (stores and products per category per date) generated rather than hand-assembled.
- Keep one source card per source (`assets/source-card.schema.json`). The daily crawl reads it, and the methods and coverage tables are generated from the set of cards.
- Record parser versions when a site layout changes; the wiki's `concepts/ai-scraping-assistants.md` covers detecting and repairing broken selectors.
- For incremental crawls, store each price row with `observed_at` and `carried_forward` (true when the product was skipped because its sitemap `<lastmod>` didn't move). Report the share of carried prices per source and day; a sudden rise usually means the sitemap stopped updating.
- Keep product identity separate from row order: product ID or canonical URL first, normalised name + brand + package size as fallback.
- Unit-test package-size extraction on real Turkish product names ("1000 GR", "1 LT", "6x200 ML", "2'li"), including multipacks.
