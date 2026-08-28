# CLAUDE.md

## Project

Momentum Scorecard — a weekly data collection system that snapshots ~150
B2B AI and fintech startups (seed through Series A) from public sources.
A momentum scoring model will be built on top of this data later.

Read `SPEC.md` before writing any code. It is the source of truth for
schemas, collectors, and scope.

## The one rule that matters most

**Collect raw values. Never store derived ones.**

Write `employee_count: 61`. Never write `headcount_growth: 0.12`.

All rates, deltas, z-scores, and normalizations are computed at scoring
time from the stored snapshot series. This is what allows the scoring model
to be rewritten in October and applied retroactively to September's data.
If a derived value gets baked into a snapshot, that flexibility is lost
permanently and the data has to be recollected — which is impossible,
because the past can't be rescraped.

The only exceptions are the explicit week-over-week diff fields named in
`SPEC.md` (`new_since_last_week`, `closed_since_last_week`,
`changed_since_last_week`).

## Scope discipline

This phase builds **collectors only**. Do not build, even if it seems
natural or you're partway through and it feels close:

- Scoring logic, weights, normalization, or cohort z-scores
- Any UI, dashboard, HTML output, or chart
- A database of any kind — the git repo is the datastore
- Prediction logging or Brier scoring
- LLM API calls
- Wayback Machine backfill

If a scoring module is needed to satisfy an import, stub it to return
`None` and add `# STUB — see SPEC.md non-goals`.

If you believe something out of scope is genuinely necessary, say so and
explain why rather than building it.

## Conventions

- Python 3.11+, `uv` for deps, `ruff` for format and lint, line length 88
- `httpx` for HTTP, `selectolax` for HTML, `pydantic` v2 for schemas,
  `typer` for CLI, `tenacity` for retries
- Type hints on every function signature
- No async unless a collector genuinely needs concurrency; sequential with
  rate limiting is fine at this scale and much easier to debug
- Schemas live in `scorecard/models.py` and nowhere else
- Every collector subclasses the `Collector` ABC in `collectors/base.py`

## Failure handling

Collector failures are normal and expected — pages move, APIs change,
companies have no public GitHub. The system must degrade, never crash.

- One collector failing must never abort the run for that company
- One company failing must never abort the run for the other 149
- Every collector always appears in the snapshot with a `status` of `ok`,
  `error`, `skipped`, or `partial` — never omit a key. Distinguishing
  "we didn't collect this" from "this doesn't exist" matters at scoring time.
- Error messages go in the snapshot's `error` field, not just the logs
- Retry with exponential backoff (tenacity), 3 attempts, then give up and
  record the failure

## Politeness — non-negotiable

- `User-Agent` on every request identifying the project with a real email
- SEC EDGAR specifically requires this and enforces 10 req/sec
- Respect `robots.txt` via `urllib.robotparser`
- Max 1 request/second per domain
- Only public, unauthenticated pages. Never authenticate to scrape.
- LinkedIn goes through the Apify actor only. Do not write a direct
  LinkedIn scraper.

## Data integrity

- `data/` is append-only in spirit. Rerunning a week overwrites that week's
  snapshot; it never touches prior weeks.
- Never modify `slug`, `added`, or `cohort` on an existing universe entry.
  CI enforces this. New entries only.
- Raw HTML is archived gzipped only when its content hash changed, to keep
  the repo small while preserving re-parseability.

## Testing

Test against a fixture set of 10 companies committed to `tests/fixtures/`,
with recorded HTTP responses. Do not hit live endpoints in tests.

The parsers that most need tests are the ATS title taxonomy (department and
seniority classification) and the EDGAR Form D XML parser. Both encode
judgment that will be wrong in interesting ways, and both feed the most
important signals.

## When you're unsure

Ask rather than guess, especially about:
- Whether a field belongs in the snapshot or is derived
- Whether a data source is being scraped politely
- Anything that expands scope beyond `SPEC.md`

Prefer boring, obvious code. This has to still run in eighteen months with
no maintenance.
