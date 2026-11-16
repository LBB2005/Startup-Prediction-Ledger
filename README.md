# Momentum Scorecard

A falsifiable forecasting record on ~150 seed-to-Series-A B2B AI and fintech
companies. The method is public, the predictions are timestamped before the
outcomes are known, and the score publishes with the misses still in it.

> ### Status: design only. Nothing has been collected.
>
> This repository is a specification and two design documents. There are no
> collectors, no `companies/universe.yaml`, no snapshots, no predictions and no
> scores. The documents were written in August 2026 for a September start;
> collection has not begun. Read this as a plan, not a result.

---

## The idea

Public data says a lot about a private company if you watch it weekly instead of
once. Job boards show whether a team is hiring its first senior salesperson.
Pricing pages show when a company decides to sell to enterprises. Changelogs and
commits show whether it is still shipping. SEC Form D filings show when money
came in.

None of that is a secret. The work is in collecting it on a schedule, storing it
raw, and committing to a prediction **before** the answer exists — then
publishing how often those predictions were right.

The research question is a horse race: **which public signal is the best leading
indicator that a company is about to start selling?** Five families of signal
compete — GTM hiring, pricing moves, hiring thresholds, shipping velocity and
enterprise-readiness — each scored on how well it predicts its own claim
resolving.

## What is in this repository

| File | What it is |
| --- | --- |
| [`SPEC.md`](SPEC.md) | The collection system: snapshot schema, the six collectors, the universe file, the CLI, the scheduled workflows. Scope is deliberately collectors-only. |
| [`docs/superpowers/specs/2026-08-28-prediction-ledger-design.md`](docs/superpowers/specs/2026-08-28-prediction-ledger-design.md) | The ledger layer: what a prediction record is, how claims resolve without human judgment, the model-versus-human head-to-head, and the three integrity mechanisms. |
| [`docs/superpowers/plans/2026-08-29-weekend-1-collectors.md`](docs/superpowers/plans/2026-08-29-weekend-1-collectors.md) | The task-by-task implementation plan for the first four collectors. |
| [`CLAUDE.md`](CLAUDE.md) | Working rules. One matters more than the rest: store raw values, never derived ones. |

## The one rule the whole design rests on

**Collect raw values. Never store derived ones.**

A snapshot records `employee_count: 61`, never `headcount_growth: 0.12`. Every
rate, delta and normalisation is computed later, at scoring time, from the series.

That is what lets the scoring model be rewritten a month later and applied
retroactively to everything already collected. The alternative loses that
permanently, and the past cannot be re-scraped — a week not collected is gone.

## How it is meant to stay honest

Three mechanisms, each aimed at a specific way a forecasting record gets
flattered.

**The cohort is frozen.** A company's `slug`, `added` date and `cohort` are
permanent and enforced by CI. Entries can be added, never edited and never
deleted. You cannot quietly add a company after it has started doing well.

**The predictions are Bitcoin-anchored.** Each week a manifest — the commit SHA,
a hash of every snapshot, a hash of every open and new prediction, and a
**monotonic cumulative count of all predictions ever made** — gets an
OpenTimestamps proof. Git commit dates are an environment variable and prove
nothing. The cumulative count is the load-bearing part: stamping predictions
individually would still allow making forty calls and publishing the twelve that
landed. Hiding a bad call would require the count to shrink, and last week's
proof says it did not. Anyone can check with `ots verify`.

**Resolution is a pure function.** Each claim type resolves from the snapshot
series by code, with no human judgment anywhere in the path. A claim that
becomes unresolvable — because a collector failed for that company all window —
is marked `expired` and reported, never silently dropped.

There is also no mechanism for skipping the hard questions. The model generates
the weekly slate; every claim on it gets answered, and a skip is recorded as a
visible skip.

## What it will not be able to tell you

Stated here rather than left to be discovered.

- **No revenue.** Private seed-stage companies have none that is public, and
  estimated figures would put an indefensible number in the ledger. It is a
  named blind spot, not an oversight.
- **Intent, not evidence.** Hiring a salesperson is a company *saying* it will
  sell. These signals do not prove customers exist.
- **Form D is thin.** It carries offering amount and security type — not
  valuation, not the round label, not a clean priced-versus-SAFE distinction.
- **GitHub is capped and sparse.** Ten most-recently-pushed repositories per
  org, and many companies have no public code at all; expect roughly half
  coverage, skewed against AI application and fintech companies.
- **Headcount is third-party.** It comes from a scrape and can lag reality.
- **The universe skews AI**, roughly 80/20. Fintech here is descriptive colour.
  No fintech finding should be promised.

## Status, honestly

The design is complete and the implementation is not started. The first
prediction has not been logged.

The schedule in the design document was built around a mid-September 2026 start,
and the design notes something that is worth repeating: the two clocks in this
project cannot be restarted. A week that is not collected cannot be recovered
afterwards, and a prediction logged late has less time to resolve. Whatever
record this project ends up with begins from the week collection actually
begins, not from the date on these documents.

Publishing the design before any of the results exist is deliberate. The
commitment in the spec is that when this is published, everything publishes —
including the early bad output.

## Politeness

Every request identifies itself with a real contact address, read from
`SCRAPER_CONTACT_EMAIL` and never hardcoded. `robots.txt` is respected, requests
are capped at one per second per domain, and only public unauthenticated pages
are read. Nothing is ever logged into in order to scrape it.
