# SPEC.md — Momentum Scorecard, Weekend 1 (Collectors Only)

## What this is

A weekly data collection system that snapshots ~150 B2B AI and fintech
startups (seed through Series A) from public sources, so that a momentum
scoring model can be built on top of it later.

**Weekend 1 ships data collection and nothing else.** The clock is only
running on collection: scoring logic written in October can be applied
retroactively to everything collected in September, but data not collected
in September is gone permanently.

## Non-goals for Weekend 1

Do not build any of these. They are explicitly out of scope:

- Scoring logic, weights, or normalization
- Any web UI, dashboard, or chart
- A database (Postgres, SQLite, anything). The git repo is the datastore.
- Prediction logging or Brier scoring
- LLM calls of any kind
- Backfill from the Wayback Machine (that's a later phase)

If a scoring function is written, it must be a stub returning `None`.

## Success criteria

Weekend 1 is done when all of these are true:

1. `python -m scorecard.collect --all` runs end to end and writes one
   snapshot file per company for the current ISO week.
2. A GitHub Actions workflow runs that command every Monday at 09:00 UTC
   and commits the results.
3. A second workflow OpenTimestamps-stamps each data commit and commits
   the `.ots` proof back.
4. Running collection twice in the same week is idempotent — it overwrites
   the current week's snapshot, never duplicates.
5. `python -m scorecard.validate` reports coverage: how many of the 150
   companies got a successful pull from each collector.
6. Coverage is at least 80% for GitHub, ATS jobs, and pricing collectors.

Nothing else matters this weekend.

---

## Stack

- **Python 3.11+**, `uv` for dependency management
- `httpx` (HTTP, with retries), `selectolax` (HTML parsing — faster and
  lighter than BeautifulSoup), `pydantic` v2 (schema validation),
  `typer` (CLI), `pyyaml`, `tenacity` (backoff)
- No ORM, no framework, no async unless a collector genuinely needs it
- Formatting: `ruff` (format + lint), line length 88

## Repo layout

```
momentum-scorecard/
  companies/
    universe.yaml            # the 150 companies — hand-curated, version controlled
    universe.schema.json     # validated on every commit by CI
  data/
    snapshots/
      2026-W36/
        modal-labs.json      # one file per company per ISO week
        ramp.json
    raw/
      modal-labs/
        pricing/
          2026-W36.html.gz   # ONLY written when content hash changes
    coverage/
      2026-W36.json          # collector success/failure report
  scorecard/
    __init__.py
    models.py                # pydantic schemas — single source of truth
    universe.py              # load + validate universe.yaml
    storage.py               # read/write snapshots, hash-gated raw archiving
    cli.py                   # typer entrypoint
    collectors/
      base.py                # Collector ABC, retry + error envelope
      github.py
      ats.py                 # Greenhouse / Lever / Ashby job boards
      pricing.py
      changelog.py
      edgar.py
      linkedin.py            # Apify — the only paid collector
  .github/workflows/
    collect.yml
    stamp.yml
  RULES.md                   # public methodology + honesty commitments
  README.md
  pyproject.toml
```

---

## The universe file

`companies/universe.yaml` is hand-curated and is the only file a human
edits regularly. Every company entry:

```yaml
- slug: modal-labs                    # stable id, never changes
  name: Modal Labs
  domain: modal.com
  sector: ai-infra                    # ai-infra | ai-app | ai-model | fintech-infra | fintech-app
  stage: series-a                     # pre-seed | seed | series-a
  business_model: b2b                 # b2b only for now; field exists for later
  added: 2026-08-30
  cohort: yc-w26                      # or: form-d-rolling, manual
  sources:
    github_org: modal-labs            # null if none
    ats:
      provider: ashby                 # greenhouse | lever | ashby | null
      token: modal                    # the board identifier
    careers_url: https://modal.com/careers
    pricing_url: https://modal.com/pricing
    changelog_url: https://modal.com/docs/reference/changelog
    docs_url: https://modal.com/docs
    linkedin_url: https://linkedin.com/company/modal-labs
    cik: null                         # SEC CIK if known, else resolved by slug/name
```

**Cohort discipline matters.** The universe is frozen at creation. Adding
a company later is allowed, but the `added` date is recorded and any future
analysis must be able to exclude late additions. This prevents
survivorship bias — you must not be able to quietly add companies after
they've already started doing well.

Write a CI check that fails if an existing entry's `slug`, `added`, or
`cohort` field is modified in a diff. New entries only.

### Building the initial 150

Do this in the CLI, not by hand:

```
python -m scorecard.universe seed --source yc --batches W26,S26 --filter ai,fintech
python -m scorecard.universe seed --source edgar --min 1500000 --max 8000000 --months 12
python -m scorecard.universe resolve-sources    # auto-detects ATS provider, GitHub org, pricing URL
```

`resolve-sources` should probe each domain for the common patterns:
`/careers`, `/jobs`, `/pricing`, `/changelog`, `/docs`, and detect the ATS
by checking whether `boards-api.greenhouse.io/v1/boards/{guess}/jobs`,
`api.lever.co/v0/postings/{guess}`, or the Ashby equivalent returns 200.
Write what it finds; leave `null` where it fails and flag for manual review.
Expect to fix 30–40 entries by hand. That's normal and it's a one-time cost.

---

## Snapshot schema

One JSON file per company per ISO week. This is the contract every
collector writes into — get it right before writing any collector.

```json
{
  "slug": "modal-labs",
  "iso_week": "2026-W36",
  "collected_at": "2026-09-07T09:04:12Z",
  "schema_version": 1,
  "collectors": {
    "github": {
      "status": "ok",
      "error": null,
      "data": {
        "public_repos": 24,
        "total_stars": 8210,
        "commits_trailing_28d": 412,
        "contributors_trailing_28d": 19,
        "distinct_repos_touched_28d": 7,
        "latest_release_tag": "v1.4.2",
        "latest_release_at": "2026-09-02T00:00:00Z"
      }
    },
    "ats": {
      "status": "ok",
      "error": null,
      "data": {
        "provider": "ashby",
        "open_roles": 23,
        "by_department": {
          "engineering": 11, "sales": 6, "marketing": 2,
          "customer-success": 2, "operations": 1, "other": 1
        },
        "by_seniority": {"junior": 4, "mid": 12, "senior": 5, "exec": 2},
        "by_location": {"san-francisco": 14, "remote": 9},
        "new_since_last_week": ["Enterprise AE, West", "Solutions Engineer"],
        "closed_since_last_week": ["Backend Engineer, Infra"],
        "raw_titles": ["..."]
      }
    },
    "pricing": {
      "status": "ok",
      "error": null,
      "data": {
        "content_hash": "sha256:9f2a...",
        "changed_since_last_week": true,
        "tier_names": ["Starter", "Team", "Enterprise"],
        "tier_count": 3,
        "has_enterprise_tier": true,
        "has_public_prices": true,
        "lowest_listed_price_usd": 30,
        "mentions_contact_sales": true,
        "raw_archived_path": "data/raw/modal-labs/pricing/2026-W36.html.gz"
      }
    },
    "changelog": {
      "status": "ok",
      "error": null,
      "data": {
        "entries_trailing_28d": 9,
        "latest_entry_at": "2026-09-05T00:00:00Z",
        "docs_page_count": 187,
        "docs_page_count_changed": 4
      }
    },
    "edgar": {
      "status": "ok",
      "error": null,
      "data": {
        "cik": "0001889374",
        "filings": [
          {"form": "D", "filed": "2025-11-14", "accession": "0001889374-25-000002",
           "total_offering_amount": 16000000, "total_sold": 16000000,
           "is_amendment": false, "security_types": ["Equity"]}
        ],
        "days_since_last_form_d": 297
      }
    },
    "linkedin": {
      "status": "ok",
      "error": null,
      "data": {
        "employee_count": 61,
        "follower_count": 14820,
        "headcount_source": "apify:linkedin-company-scraper",
        "scraped_at": "2026-09-07T09:11:00Z"
      }
    }
  }
}
```

### Rules for this schema

- **Every collector always appears**, even on failure. A failed collector
  writes `{"status": "error", "error": "<message>", "data": null}`. A
  collector that isn't configured for this company writes
  `{"status": "skipped", "error": "no github_org", "data": null}`.
  Never omit a key — missing-vs-failed must be distinguishable later.
- **`status` is one of:** `ok`, `error`, `skipped`, `partial`.
- **Store raw values, never derived ones.** Write `employee_count: 61`,
  never `headcount_growth: 0.12`. All deltas and rates are computed at
  scoring time from the snapshot series. This is what lets you rewrite the
  scoring model later without recollecting.
- **One exception:** `new_since_last_week` / `closed_since_last_week` /
  `changed_since_last_week` are stored because they require comparing to
  the previous snapshot at collection time and are cheap to keep.
- Timestamps are ISO 8601 UTC with `Z`.
- Money in whole USD integers, never floats.

---

## The six collectors

Build them in this order. Each is independent — a failure in one must
never abort the run.

### 1. `ats.py` — job boards (BUILD FIRST, highest signal, free, structured)

Most B2B startups use Greenhouse, Lever, or Ashby, and all three expose
public JSON endpoints with no auth and no scraping:

- Greenhouse: `https://boards-api.greenhouse.io/v1/boards/{token}/jobs?content=true`
- Lever: `https://api.lever.co/v0/postings/{token}?mode=json`
- Ashby: `https://api.ashbyhq.com/posting-api/job-board/{token}`

This is the single most valuable collector. Hiring **mix** — the ratio of
engineering to go-to-market roles, and how it shifts over time — is the
best public proxy for a company finding product-market fit and starting to
commercialize. Get the department classification right.

Departments are inconsistent across providers, so normalize titles yourself
with a keyword ruleset in `scorecard/collectors/taxonomy.py`. Map to a
fixed set: `engineering`, `product`, `design`, `sales`, `marketing`,
`customer-success`, `operations`, `finance`, `people`, `other`. Same for
seniority: match on `intern|junior|associate` → junior,
`senior|staff|principal|lead` → senior, `head of|vp|director|chief|founding`
→ exec, else mid. Keep the ruleset in one file, keep `raw_titles` in the
snapshot so a better classifier can be applied retroactively.

For companies with no ATS, fall back to hashing the careers page and
counting `<a>` elements that look like job links. Mark `status: "partial"`.

### 2. `github.py` — product velocity (free, 5000 req/hr with a token)

Use the REST API with a personal access token in `GITHUB_TOKEN`.
For each org: list repos, then for each non-fork repo use the commit
activity or list-commits endpoint with `since=` 28 days ago.

Cap at the 10 most recently pushed repos per org to stay inside rate
limits. Note this cap in `RULES.md` — it's a real methodological limitation
and you should disclose it rather than let someone find it.

Many companies have no public GitHub. That's a legitimate `skipped`, and
it means the Product sub-score will be sparse for AI application and
fintech companies specifically. Expect roughly 50% coverage here and don't
treat the gap as a bug.

### 3. `pricing.py` — commercialization signals (free)

Fetch the pricing page. Extract visible text, normalize whitespace, hash it.
Compare against last week's hash.

Parse for: tier names (headings inside pricing card elements), whether a
tier named `Enterprise`/`Custom` exists, whether any dollar amounts are
shown, the lowest listed price, and whether "contact sales" appears.

**Archive the raw HTML gzipped only when the hash changes.** Most weeks
nothing changes, so this keeps the repo small while preserving the ability
to re-parse history with a better parser. Rough sizing: 150 companies at
~30KB gzipped, with maybe 10% changing weekly, is under 30MB/year.

Pricing pages are heavily JavaScript-rendered. Accept failure gracefully —
if the extracted text is under 200 characters, record
`status: "partial"` with the hash only. Do not add a headless browser this
weekend.

### 4. `changelog.py` — shipping cadence (free)

Try, in order: an RSS/Atom feed at the changelog URL, then a JSON feed,
then HTML parsing for date-like patterns near headings. Count entries with
a parsed date inside the trailing 28 days.

For `docs_page_count`, fetch `/sitemap.xml` and count URLs under the docs
path. This is a surprisingly good proxy for product surface area growth and
costs one request.

### 5. `edgar.py` — capital events (free, official)

SEC requires a `User-Agent` header identifying you with a real email
address, and enforces a limit of 10 requests per second. Set:

```python
HEADERS = {"User-Agent": "Momentum Scorecard <your-email@usc.edu>"}
```

Two modes:
- **Resolve:** map company name → CIK using the full company tickers/company
  index file, cached locally. Fuzzy-match with a manual override map in
  `companies/cik_overrides.yaml` for the ones it gets wrong. It will get
  plenty wrong; private company names are messy.
- **Fetch:** pull the submissions JSON for each known CIK, filter to Form D
  and D/A, parse the XML primary document for offering amount, amount sold,
  and security type.

Also run a daily index scan for *new* Form D filings matching your sector
codes — that's what feeds new companies into the rolling pool later.

Important limitation to write down in `RULES.md`: Form D gives offering
amount and security type, but **not valuation, not the round label, and not
a clean priced-vs-SAFE distinction.** Don't let downstream logic pretend
otherwise.

### 6. `linkedin.py` — headcount (Apify, ~$50/mo, BUILD LAST)

Use an Apify LinkedIn company scraper actor. Input the 150 LinkedIn URLs,
run once weekly, read the dataset, extract `employee_count` and
`follower_count`.

Isolate this behind the same `Collector` interface as everything else so
that if the actor breaks, changes shape, or gets rate-limited, only this
collector fails. Set `APIFY_TOKEN` in secrets. Cap the monthly spend in the
Apify console so a runaway loop can't drain the budget.

If it fails: employee count on the company's own careers page or About page
is a weak fallback. Mark `status: "partial"`.

---

## CLI

```
python -m scorecard.collect --all                 # all collectors, all companies
python -m scorecard.collect --only ats,github     # subset of collectors
python -m scorecard.collect --company modal-labs  # single company, for debugging
python -m scorecard.collect --dry-run             # fetch, print, write nothing
python -m scorecard.validate                      # coverage report for current week
python -m scorecard.universe add <domain>         # interactive add w/ source resolution
python -m scorecard.universe check                # schema + frozen-field validation
```

Collection must be **resumable**: if it crashes at company 90, rerunning
skips the 89 already written for this week unless `--force` is passed.

Log one line per company per collector to stdout in a parseable format.
You will be debugging this at 1am and grep is your friend.

---

## GitHub Actions

### `collect.yml`

```yaml
on:
  schedule: [{cron: "0 9 * * 1"}]   # Mondays 09:00 UTC
  workflow_dispatch:
```

Steps: checkout → setup uv → `python -m scorecard.collect --all` →
`python -m scorecard.validate` → commit `data/` with message
`data: snapshot 2026-W36` → push.

Do not let the job fail the workflow when individual collectors fail. Fail
only if **zero** companies were collected, or if overall coverage drops
below 50%, which indicates something structural broke.

### `stamp.yml`

Triggers on push to `main` when `data/**` changed. Installs the
OpenTimestamps client, runs `ots stamp` on a manifest file containing the
commit SHA plus a hash of every snapshot written that week, commits the
`.ots` proof.

This is the piece that makes the timestamps real. Git commit dates are a
settable environment variable and prove nothing; a Bitcoin-anchored
timestamp proves the data existed before a given block. Anyone can verify
it independently with `ots verify`.

Note the `.ots` proof takes a few hours to fully confirm — that's expected,
and `ots upgrade` completes it later. Add a monthly workflow that runs
`ots upgrade` across all proofs.

---

## RULES.md — write this before collecting anything

A short public methodology page. It must state:

1. Data sources used and what each one can and cannot show
2. The GitHub 10-repo cap and why it exists
3. That Form D shows offering amount but not valuation or round label
4. That the universe is frozen, additions are dated, and nothing is ever
   deleted
5. That headcount comes from a third-party scrape and may lag reality
6. That no prediction or score ever uses confidential or non-public
   information from any affiliation or employer
7. That the repo is currently unlisted, and a commitment that when it is
   published, everything publishes — including the early bad output

Point 6 is not optional. A public timestamped record is permanent, which is
the entire point, and that cuts both ways.

---

## Politeness and legal hygiene

- Identify yourself in the `User-Agent` on every request, with a real email
- Respect `robots.txt` — use `urllib.robotparser` and skip disallowed paths
- 1 request/second per domain, hard ceiling
- Cache aggressively; never refetch an unchanged page within a week
- Only public, unauthenticated pages. Never log into anything to scrape it.
- LinkedIn data goes through Apify's actor, which is the paid intermediary
  handling that relationship. Don't write your own LinkedIn scraper.

---

## Build order for the weekend

**Saturday morning (3h)** — `pyproject.toml`, `models.py` with full
pydantic schemas, `storage.py`, `universe.yaml` with 10 hand-entered
companies, `Collector` base class. Write the schema first and resist the
urge to start scraping; every hour spent on the schema saves three later.

**Saturday afternoon (4h)** — `ats.py` end to end, including the title
taxonomy. Get all three ATS providers working against your 10 test
companies. This is the highest-value collector; give it the most time.

**Saturday evening (2h)** — `github.py`, then `collect.yml` committing to
the repo. Get one full automated run green.

**Sunday morning (3h)** — `pricing.py` and `changelog.py`, plus the
hash-gated raw archiving.

**Sunday afternoon (3h)** — `edgar.py` including CIK resolution and the
override map. Budget more time than feels reasonable; name matching for
private companies is genuinely annoying.

**Sunday evening (3h)** — `linkedin.py` via Apify, `stamp.yml`, `RULES.md`,
then expand the universe from 10 to 150 using the seeding commands and fix
the resolution failures by hand while watching something.

**Ship condition:** a green scheduled run producing 150 snapshot files with
≥80% coverage on ATS, GitHub, and pricing. Scoring is next weekend.
