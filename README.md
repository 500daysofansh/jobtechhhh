# Sarkari Jobs Aggregator (CS/IT/B.Tech/M.Tech/Diploma Focus) — Architecture & Roadmap

## 1. Product Summary

A niche job-aggregator site for technical government jobs, structured around four content pillars per exam/job:

```
Upcoming → Apply → Prepare → Result/Admit Card
```

Monetization: AdSense + direct sponsored placements + affiliate course links + (later) premium tier and B2B data licensing.

---

## 2. High-Level System Architecture

```
                        ┌─────────────────────────┐
                        │   SCHEDULER (Cron/Beat)   │
                        │  runs every 1–6 hours     │
                        └────────────┬──────────────┘
                                     │
                 ┌───────────────────┼───────────────────┐
                 ▼                   ▼                   ▼
        ┌────────────────┐ ┌────────────────┐  ┌────────────────┐
        │ Scraper Worker  │ │ Scraper Worker  │  │ Scraper Worker  │
        │ (Official Site A)│ │(Official Site B)│  │ (Aggregator ref)│
        └────────┬────────┘ └────────┬────────┘  └────────┬────────┘
                 │                   │                    │
                 └───────────────────┼────────────────────┘
                                     ▼
                        ┌─────────────────────────┐
                        │   RAW STAGING TABLE       │
                        │ (unprocessed scraped data)│
                        └────────────┬──────────────┘
                                     ▼
                        ┌─────────────────────────┐
                        │  NORMALIZER + DEDUP        │
                        │  (fuzzy match on title+org)│
                        └────────────┬──────────────┘
                                     ▼
                        ┌─────────────────────────┐
                        │  QUALIFICATION CLASSIFIER │
                        │ (regex rules + LLM fallback)│
                        └────────────┬──────────────┘
                                     ▼
                        ┌─────────────────────────┐
                        │   MAIN DATABASE (Postgres) │
                        │ jobs / prepare / upcoming   │
                        └────────────┬──────────────┘
                                     ▼
                 ┌───────────────────┼───────────────────┐
                 ▼                   ▼                   ▼
        ┌────────────────┐ ┌────────────────┐  ┌────────────────┐
        │  Backend API     │ │ Telegram/WhatsApp│ │  Admin Panel    │
        │ (REST/GraphQL)   │ │  Alert Bot        │ │ (manual review) │
        └────────┬────────┘ └────────────────┘  └────────────────┘
                 ▼
        ┌────────────────┐
        │  Frontend (Next.js)│
        │  SSR/ISR for SEO   │
        └────────┬────────┘
                 ▼
        ┌────────────────┐
        │   CDN (Cloudflare)│
        └────────┬────────┘
                 ▼
              Users
```

---

## 3. Tech Stack Recommendation

| Layer | Recommended | Why |
|---|---|---|
| Frontend | **Next.js (React)** | SSR/ISR critical for SEO on job pages; fast, mobile-first |
| Styling | Tailwind CSS | Fast to build, lightweight for slow connections |
| Backend API | Node.js (Express/Fastify) or Django REST | Either fine; Django REST if you want admin panel out of the box |
| Database | **PostgreSQL** | Structured relational data (jobs, categories, dates) fits well |
| Scraping | Python (`httpx`/`requests` + `BeautifulSoup`, `Scrapy` for scale, `Playwright` for JS-heavy pages) | Best scraping ecosystem |
| Task Scheduling | Celery + Redis (Python) or BullMQ (Node) | Reliable recurring scrape jobs |
| Search/Filter | Postgres full-text search initially → Elasticsearch/Meilisearch if scale grows | Avoid over-engineering early |
| Hosting | Vercel (frontend) + Railway/Render/DigitalOcean (backend+DB) initially | Cheap to start, scales later |
| CDN/Caching | Cloudflare | Free tier, huge help for SEO + speed + DDoS protection |
| Alerts | WhatsApp Business API (or unofficial Telegram Bot API — official/safer for WhatsApp) | Retention channel |
| Ads | Google AdSense + Google Ad Manager (for sponsor rotation) | Standard setup |
| Analytics | Google Analytics 4 + Search Console | Track SEO performance |

---

## 4. Database Schema (Core Tables)

```sql
-- Sources you scrape from
sources (
  id, name, base_url, type ENUM('official','aggregator'),
  last_scraped_at, is_active
)

-- Raw staging (before cleaning)
raw_scraped_jobs (
  id, source_id, raw_html_or_json, scraped_at, processed BOOLEAN
)

-- Core jobs table
jobs (
  id,
  title,
  organization,
  category ENUM('technical','non_technical','mixed'),
  qualification_tags TEXT[],   -- ['B.Tech','Diploma','CS','IT','M.Tech']
  vacancy_count INT,
  location TEXT,
  post_date DATE,
  last_apply_date DATE,
  exam_date DATE NULL,
  status ENUM('upcoming','open','closed','result_declared'),
  official_apply_link TEXT,
  official_notification_pdf TEXT,
  source_id FK,
  slug TEXT UNIQUE,           -- for SEO-friendly URL
  created_at, updated_at
)

-- Prepare content, one-to-one/many with jobs
prepare_content (
  id, job_id FK,
  syllabus TEXT,
  exam_pattern TEXT,
  previous_papers_links TEXT[],
  cutoff_data JSONB,          -- {year: {category: score}}
  notes_content TEXT,
  recommended_courses JSONB   -- [{title, affiliate_link, sponsor_id}]
)

-- Upcoming/predicted jobs
upcoming_predictions (
  id, exam_name, organization,
  expected_month_year TEXT,
  based_on_years INT[],
  confidence ENUM('high','medium','low'),
  notify_subscribers_count INT
)

-- User accounts (for premium/alerts)
users (
  id, email, phone, qualification_pref TEXT[],
  location_pref TEXT, is_premium BOOLEAN, created_at
)

-- Alert subscriptions
subscriptions (
  id, user_id FK, job_id FK NULL, upcoming_id FK NULL,
  channel ENUM('email','telegram','whatsapp'), created_at
)

-- Sponsors/advertisers
sponsors (
  id, name, contact_email, plan_type, active_from, active_to
)

sponsor_placements (
  id, sponsor_id FK, job_id FK NULL, page_type ENUM('job','prepare','homepage'),
  banner_url, target_link, impressions_count, clicks_count
)
```

---

## 5. Scraping Pipeline (Detail)

1. **Source config table** — each official portal (SSC, UPSC, state PSCs, PSUs, IBPS, Railways) has its own scraper module since HTML structures differ.
2. **Scheduler** triggers scrapers every 1–6 hrs depending on source update frequency.
3. **Raw staging** — always store raw scraped payload first (HTML snapshot or parsed JSON) before processing — lets you re-parse without re-scraping if your parser had a bug.
4. **Normalizer** — standardizes date formats, organization names, dedupes near-identical postings (fuzzy match title+org+date within a threshold).
5. **Qualification Classifier**:
   - Pass 1: regex/keyword rules (`B\.?Tech`, `Diploma`, `M\.?Tech`, `Computer Science`, `IT`, `Electronics`, `B\.?E\.?`)
   - Pass 2: for ambiguous postings (no clear regex match but plausible), send notification text to an LLM classification call — cheap at this volume, much more accurate than keyword-only
   - Output: qualification_tags array on the job
6. **Manual review queue** — flag low-confidence classifications for a human (you, initially) to approve before publishing. Important early on to avoid embarrassing misclassifications.
7. **PDF parsing** — many official notifications are PDFs; use `pdfplumber`/`PyMuPDF` to extract structured fields (dates, vacancy tables, eligibility) automatically where possible.

---

## 6. Frontend Structure (SEO-First URL Design)

```
/                                → homepage (latest + filters)
/jobs/[slug]                     → individual job page (Apply/Prepare/Result tabs)
/category/b-tech                 → all B.Tech jobs
/category/diploma                → all Diploma jobs
/upcoming                        → upcoming/predicted notifications
/upcoming/[exam-slug]             → e.g. /upcoming/ssc-cgl-2027
/exam/[exam-name]/prepare         → syllabus/pattern/notes hub per recurring exam
/results                          → results + admit cards
/telegram                         → channel signup landing page
```

Each job page includes:
- JSON-LD `JobPosting` schema markup (critical for Google Jobs integration — huge free traffic source)
- Unique, human-written or LLM-drafted-then-edited descriptions (never copy competitor text)
- Tabs: Apply | Prepare | Result (single URL, better for SEO than splitting into 3 pages)

---

## 7. Roadmap (Phased)

### Phase 0 — Planning (Week 1–2)
- Finalize list of 15–20 official source sites to scrape (prioritize technical-heavy recruiters: SSC, PSU's, Railways, State PSCs, DRDO, ISRO, BEL etc.)
- Define qualification taxonomy (exact tags you'll use)
- Set up repo, hosting accounts, domain

### Phase 1 — MVP Scraper + Database (Week 3–5)
- Build scrapers for top 5 sources
- Build normalizer + basic regex classifier
- Populate Postgres with first batch of jobs
- Manual QA on classification accuracy

### Phase 2 — MVP Website (Week 6–9)
- Next.js frontend: homepage, job listing page, category filters
- Apply tab only (Prepare/Upcoming come later)
- Deploy to Vercel + Railway, connect domain
- Submit sitemap to Google Search Console, add JobPosting schema

### Phase 3 — Telegram/WhatsApp Alerts (Week 10–11)
- Build simple bot that posts new jobs matching CS/IT/B.Tech/Diploma tags
- This becomes your primary retention/growth channel while SEO ramps up (SEO takes months)

### Phase 4 — Prepare Section (Week 12–15)
- Syllabus/pattern extraction pipeline (PDF parsing + manual cleanup)
- Previous year papers linking
- First affiliate/course placements (start with 2-3 affiliate programs, e.g. Testbook/Adda247)

### Phase 5 — Upcoming/Predicted Jobs (Week 16–18)
- Historical pattern tracker for recurring exams
- "Notify me" opt-in on upcoming pages

### Phase 6 — Monetization Scale-Up (Month 5+)
- Apply for AdSense once you have ~30-50 quality pages and real traffic
- Start direct outreach to coaching institutes/teachers for sponsored slots
- Add premium tier (ad-free + early alerts) once you have retained users

### Phase 7 — Growth & Scale (Month 6+)
- Expand scraper sources to 40-50+ official portals
- Add mock tests/quizzes
- Consider Elasticsearch if job volume/search complexity grows
- Explore B2B data licensing to ed-tech companies

---

## 8. Key Risks & Mitigations

| Risk | Mitigation |
|---|---|
| Scraper breaks when source changes HTML | Modular per-source scrapers, alerting on scrape failures, prioritize official sources over aggregators |
| Copyright issues from reused text | Always rewrite descriptions; use facts/data, not copied prose |
| AdSense rejection (thin/duplicate content) | Ensure enough original value-add (Prepare content, filtering, own writing) before applying |
| Wrong "upcoming" dates causing user complaints | Clearly label as "expected/tentative," cite basis |
| Low ad RPM in this niche | Diversify early — affiliate + sponsor deals + Telegram monetization, don't rely on AdSense alone |
| SEO takes long to rank against established competitors | Niche down hard (CS/IT/B.Tech specifically) rather than competing broadly; build Telegram audience in parallel as owned channel |

---

## 9. Rough Cost Estimate (Early Stage)

- Domain: ~₹800–1200/year
- Hosting (Vercel + small DB instance): ~$0–20/month to start (many free tiers)
- LLM API calls for classification (low volume): a few $ per month
- WhatsApp Business API (if used at scale): has per-message costs; Telegram is free
- AdSense: free to join, revenue-share only

Total to get MVP live: **minimal cash cost**, mostly your time.
