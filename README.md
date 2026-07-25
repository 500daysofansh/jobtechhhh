# Sarkari Jobs Aggregator — Phase 1

Database schema + first scraper (SSC), per the architecture roadmap.

## Prerequisites

- Node.js 18+
- PostgreSQL 14+ running locally (or Docker)

## Setup

```bash
npm install
npx playwright install chromium   # downloads the browser binary Playwright drives

cp .env.example .env
# edit .env with your actual DATABASE_URL

createdb sarkari_jobs             # or use your Postgres GUI of choice
npm run db:migrate                # applies db/migrations/*.sql in order, seeds SSC as a source
```

## Before running the SSC scraper

`scraper/sources/ssc.js` has two strategies: capturing the site's internal
JSON API (fast, reliable) and falling back to DOM scraping (slower, breaks
easily). I could not load ssc.gov.in from my sandbox to inspect it directly,
so **you need to do one manual check first**:

1. Open `https://ssc.gov.in/notice` in Chrome
2. DevTools → Network tab → filter to Fetch/XHR → reload the page
3. Find the request returning the notice list as JSON, note its URL and
   field names
4. Update `API_ENDPOINT_GUESS` and `mapApiResponse()` in `scraper/sources/ssc.js`
   accordingly — or just run it as-is first, since the scraper listens for
   *any* response matching `/api/.../notice.../` and will print what it
   captures so you can adjust the mapping from real data.
5. If DOM fallback triggers, also verify the CSS selectors in `scrapeDom()`
   against the live page (right-click a notice row → Inspect).

## Run the scraper

```bash
npm run scrape:ssc
```

This will:
- Launch headless Chromium via Playwright
- Try to capture the notice-list API response; fall back to DOM scraping
- Insert results into `raw_scraped_jobs` (staging — nothing is published yet)
- Log a `scrape_runs` row so you can see success/failure history

Check what landed in staging:

```sql
SELECT raw_title, raw_url, scraped_at FROM raw_scraped_jobs ORDER BY scraped_at DESC LIMIT 20;
```

## What's NOT built yet (next steps, in order)

1. **Normalizer + classifier** — moves rows from `raw_scraped_jobs` →
   `jobs`, applying the qualification-tag regex rules, dedup, and setting
   `review_status = 'pending_review'`
2. **Admin review endpoint** — approve/reject queued jobs before they go
   live (schema already has `review_status` for this)
3. **Public API** (`api/`) — read-only endpoints for the frontend
4. **Scheduler** — cron/systemd timer to run `npm run scrape:all`
   periodically instead of manually
5. **More sources** — add scraper modules under `scraper/sources/` and
   register them in `scraper/runner.js`'s `SCRAPER_REGISTRY`, plus a row
   in the `sources` table

## Project structure so far

```
db/
  client.js              Postgres connection pool
  migrations/001_init.sql  sources, raw_scraped_jobs, jobs, scrape_runs
scraper/
  runner.js              orchestrates: source config -> scraper -> staging insert
  sources/ssc.js          SSC-specific scraper (API-capture + DOM fallback)
  util/hash.js            exact-dup fingerprinting before insert
scripts/migrate.js        applies SQL migrations in order
```
