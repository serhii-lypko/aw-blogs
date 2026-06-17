# Blog crawler agent

You are a fully autonomous agent that crawls a fixed set of blogs, extracts
posts published since the last crawl, and writes a formatted report of each.

## Source of truth for the window

`checkpoint.yaml` holds `last_crawl_at`. A `fetch` collects every post whose
published date is **strictly after `last_crawl_at`** and **at or before the
moment the run started** (`run_started_at`). The typical cadence is weekly, but
the window is defined entirely by the checkpoint — never by a hardcoded 7 days.

Capture `run_started_at` once, in UTC, at the very start of the run. Use that
same value as the upper bound for every blog, so a post published mid-run is
never silently skipped.

## Files

`blogs.yaml`: resources grouped under named categories. Each category key (e.g.
`programming`, `geopolitics`, `science`) maps to a list of resources.

```yaml
resources:
  programming:
    - rss: https://example.com/atom.xml   # optional; preferred source
      fallback: https://example.com/blog/ # required; homepage to scrape if no/failed rss
      alias: example                      # required; display name + grouping key
  science:
    - fallback: https://no-feed.example/  # rss may be absent; fallback is then the only source
      alias: another
```

- `rss` may be absent. Then the `fallback` HTML page is the only source.
- `fallback` is always required.
- `alias` is unique across all categories and used as the blog's group header and stable id.
- Category order, and resource order within a category, follow `blogs.yaml`.

`checkpoint.yaml`:

```yaml
last_crawl_at: 2025-06-14T00:00:00Z   # ISO 8601, UTC, always with Z
```

## Algorithm

For each resource (process independently; one failure must not abort others):

1. If `rss` is present, fetch and parse it. For each entry read the published
   (or updated, if no published) timestamp **and the canonical post URL**.
2. If `rss` is absent, or the fetch/parse fails, scrape `fallback`: locate the
   post listing, and for each post extract its URL, title, and published date.
   If a date is only relative ("3 days ago"), resolve it against
   `run_started_at`. If no date can be determined for a post, skip it and record
   it under skipped (do not guess it into range).
3. Keep posts where `last_crawl_at < published <= run_started_at`.
4. For each kept post, produce a **generated** summary in your own words from the
   post body — roughly 4–6 sentences (~1000–1400 chars), enough to convey the
   post's argument, key claims, and conclusions, not just its topic. This is a
   summary, not a verbatim excerpt — do not quote the lede.
5. Sort kept posts within a blog newest-first.

After all resources succeed at least at the run level, set
`last_crawl_at = run_started_at` and write `checkpoint.yaml`. Write the
checkpoint **only if the run completed**; if the whole run aborts, leave the old
checkpoint untouched so the next run re-attempts the same window.

## Fetch discipline

- Process resources strictly sequentially, in `blogs.yaml` order (category by
  category, resource by resource). Never fetch in parallel.
- Per resource, at most 3 fetch attempts total, and only for *transient*
  failures: connection timeout, reset, or 5xx. Back off briefly between attempts.
- Deterministic failures — 404/410, malformed feed or HTML that fails to parse,
  no datable posts — are not retried. Abandon the resource on the first such
  failure.
- On exhausting 3 attempts or hitting a deterministic failure, abandon the
  resource, record it under skipped/failed, and move to the next. Do not block or
  loop.
- Global ceiling: cap the whole run at 30 total fetch attempts across all
  resources. On reaching it, stop crawling, write no checkpoint, and report what
  completed plus what was skipped.

## Failure handling

- A blog that 404s, times out, returns malformed feed/HTML, or yields no datable
  posts is logged and skipped — the run continues.
- At the end, include a short "Skipped / failed" section listing each failed
  alias and the reason. Empty in the happy path.
- Advancing the checkpoint despite some blogs failing is acceptable (incremental,
  best-effort). A run-level abort (global ceiling) writes no checkpoint; a single
  dead blog does not block the checkpoint.

## Crawler etiquette

Send a descriptive User-Agent, respect `robots.txt`, and space requests so a
single run doesn't hammer any host.

## Output

Use the `/pdf` skill to produce the report. Render it as a nicely formatted PDF
written to `./reports/`, filename `<run date>.pdf` (e.g. `2026-06-17.pdf`, from
`run_started_at`). Do not overwrite an existing file — if it exists, suffix
`-2`, `-3`, etc. Create `./reports/` if absent.

Document structure:

- Header: the date range `<min published date> - <max published date>` across
  all kept posts, formatted `DD mon` (e.g. `15 jun - 21 jun`). If only one post,
  both ends are equal.
- Then one section per category that has posts, in `blogs.yaml` order, each under
  a category heading.
- Within a category, one block per blog that has posts, in `blogs.yaml` order:
  the alias as a sub-header, followed by its posts as a list, newest-first.
- Each post: the title as a clickable link to the original post, followed by the
  summary.
- Categories and blogs with no posts in range are omitted.

Use clean, readable typography — clear heading hierarchy (category > blog >
post), comfortable line spacing, and live hyperlinks on titles.

## The interface

This project is driven by claude/codex agents. Parse the raw command and act.

## Commands

- `fetch` (also the default when invoked with no argument): run the algorithm
  above and write the report for the current window.
- `reset`: set `last_crawl_at` to `1970-01-01T00:00:00Z` and write
  `checkpoint.yaml`. The next `fetch` then collects all available history.
- `back-1w`: move the checkpoint backward 1 week.
- `back-2w`: move the checkpoint backward 2 weeks.
- `back-1m`: move the checkpoint backward 1 month.
- `back-3m`: move the checkpoint backward 3 months.
- `help`: list available commands.