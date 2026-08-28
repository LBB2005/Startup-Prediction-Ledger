# Prediction Ledger — Design

**Date:** 2026-08-28
**Status:** Approved
**Repo:** `LBB2005/Startup-Prediction-Ledger` (private; publish decision deferred)

---

## 1. Purpose

Produce a **falsifiable, cryptographically-timestamped forecasting record** on
~150 seed-stage B2B AI and fintech companies, such that by January 2027 the
following claim is true and independently verifiable:

> "I have published timestamped predictions on 150 seed-stage companies since
> September 2026. Public methodology, public Brier score, including the calls I
> got wrong."

The collection system described in `SPEC.md` is the substrate. It is necessary
but not sufficient — on its own it demonstrates engineering, not judgment. The
ledger layer is what makes the project a claim about judgment.

### Why the schedule is aggressive

Two clocks run independently and neither can be restarted:

1. **Collection.** Snapshots not taken in September cannot be recovered. The
   past cannot be rescraped.
2. **Prediction.** A Brier score requires resolved outcomes. A prediction first
   logged in March has no resolution by recruiting season.

The second clock is the binding constraint and is why prediction infrastructure
is being designed now rather than deferred.

### The horizon constraint

Seed to Series A resolves in 12–24 months. Over a four-month window the base
rate is roughly 5–8%, yielding ~10 resolved events across 150 companies — not
enough to support any claim.

Therefore the ledger is **two-tier**:

- **Tier 1 (headline):** claims resolving in 4–12 weeks, auto-resolvable from
  the collectors. These produce a scored record by January.
- **Tier 2 (long-horizon):** funding-outcome claims logged now on a 12–18 month
  clock. Near-zero marginal cost; becomes the story in 2027–28.

---

## 2. Architecture

```
scorecard/    # collection — per SPEC.md, unchanged
ledger/       # predictions, resolution, scoring — new
```

`ledger/` imports from `scorecard/`. **Never the reverse.**

This preserves the core rule in `CLAUDE.md`: snapshots remain raw-values-only in
perpetuity, and every derived quantity lives in `ledger/`, recomputed from the
snapshot series on demand. The property that the scoring model can be rewritten
later and applied retroactively survives intact.

---

## 3. The prediction record

Every prediction is a **machine-evaluable predicate over future snapshots**.

```json
{
  "claim_id": "2026-W37-ramp-gtm-001",
  "record_id": "2026-W37-ramp-gtm-001:model",
  "created_at": "2026-09-14T09:00:00Z",
  "iso_week": "2026-W37",
  "slug": "ramp",
  "claim_type": "gtm_inflection",
  "claim": "ats.by_department.sales with seniority in (senior,exec) >= 1 by 2026-11-30",
  "resolves_by": "2026-11-30",
  "predictor": "model",
  "probability": 0.65,
  "model_version": "v0.1.0",
  "config_version": "thesis-v0.1.0",
  "rationale": null,
  "features_at_creation": {},
  "calibration_phase": true,
  "quota_filled": false,
  "status": "open",
  "outcome": null,
  "resolved_at": null,
  "resolution_evidence": null
}
```

Notes:

- `features_at_creation` freezes the inputs the predictor saw, for audit.
- `model_version` and `config_version` are frozen per record. Rewriting either
  later never mutates an existing prediction.
- `claim_id` identifies the *question*; `record_id` is `{claim_id}:{predictor}`
  and is the unique key. Every claim therefore has exactly two records — one
  `model`, one `human` — which is what makes the comparison paired rather than
  two independent samples. Resolution writes the same `outcome` to both.

### Claim types

| Type | Resolves from | Horizon |
|---|---|---|
| `gtm_inflection` | ATS department + seniority taxonomy | 8–12 wk |
| `pricing_move` | pricing content hash + tier parser | 4–12 wk |
| `hiring_threshold` | ATS `open_roles` | 4–8 wk |
| `shipping_velocity` | GitHub releases + changelog entries | 8–12 wk |
| `enterprise_motion` | docs sitemap enterprise-readiness flags | 8–12 wk |
| `funding_event` (Tier 2) | EDGAR Form D | 12–18 mo |

---

## 4. Resolution

Each claim type implements a pure function:

```python
def resolve(claim: Claim, series: SnapshotSeries) -> bool | None: ...
```

`None` means not yet resolvable. Runs automatically after each weekly
collection. **No human judgment touches resolution.** This is the integrity core
of the project; it is the most heavily tested code in the repo.

A claim whose `resolves_by` passes while still unresolvable — e.g. the collector
failed for that company for the whole window — is marked `expired`, never
silently dropped, and reported separately in coverage.

---

## 5. Predictors

### Model v0.1 — deliberately trivial

v0.1 predicts the **cohort base rate and nothing else.** No features, no weights.

This is not a placeholder to be apologised for. With zero training data, any
"sophisticated" v0.1 is invented numbers wearing a lab coat. A base-rate
baseline is the correct null hypothesis, and it means the discretionary picks
compete against an honest benchmark from day one.

Model v0.2 — around November, once resolved outcomes exist — learns weights from
those outcomes. Every v0.1 prediction remains permanently attributed to v0.1.

### Human

Discretionary probability on every claim in the weekly slate. Optional free-text
`rationale`.

### The separation that makes this work

> **The thesis decides which questions get asked. The data decides the answers.**

The model's probability is derived from empirical base rates only — never from
the user's stated beliefs. If the model encoded the thesis, it would be the
human with extra steps, and the head-to-head would measure nothing.

This is statistically safe: a thesis-biased slate does not corrupt the Brier
comparison, because both predictors are scored on **identical claims**. Tuning
the thesis config later therefore invalidates no prior score.

---

## 6. Slate generation

Five decisions per week. Steps 1, 2, 3 and 5 encode the thesis; step 4 does not.

1. **Eligibility** — company has required collector coverage; no open claim of
   the same type; claim not already resolved.
2. **Trigger** — a rule over the snapshot series fires.
3. **Claim construction** — trigger determines claim type, threshold, deadline.
4. **Probability** — base rate (model) and discretionary (human).
5. **Stratified selection** — allocate a floor of 3 claims per trigger family,
   then fill remaining slots by readiness score. 20 claims total.

### Triggers, phased

Three of the four selected trigger families are **delta triggers** requiring a
trailing baseline. On 2026-09-14 no history exists. Triggers are therefore
split so the first-prediction date holds.

**Phase A — level triggers, live week 1.** Read a single snapshot; no history.

| Trigger | Fires when | Generated claim |
|---|---|---|
| Primed, unfired | eng share ≥75%, zero senior GTM roles, `open_roles` ≥3 | posts first senior GTM role by +10wk |
| Fresh capital | Form D filed <180d ago | `open_roles` up ≥40% by +12wk |
| Pricing headroom | public prices, no enterprise tier | adds enterprise/custom tier by +12wk |
| Dark pricing | pricing page exists, no public prices | publishes a price by +12wk |
| Enterprise motion | SSO/SAML/SCIM/audit-log docs present | adds enterprise tier by +12wk |

EDGAR is historical, so the capital trigger works on day one.

**Phase B — delta triggers, live week 5**, once a trailing window exists: mix
shift, hiring surge, headcount acceleration, ship spike, and the history-aware
form of GTM ignition.

Weeks 1–4 are not wasted: they generate real Phase A claims *and* accumulate the
baseline that unlocks Phase B.

### Ranking objective — "primed but not yet fired"

The target is the **moment of commercialization**. A company with 25 engineers,
zero AEs, a Form D 90 days ago and no enterprise tier is at that moment. A
company with six AEs and an enterprise tier has passed it — there is nothing
left to predict. The score therefore rewards capacity and fuel, and penalises
evidence the move already happened:

```
readiness =  w1 * eng_share
           + w2 * capital_recency        # Form D within 180d
           + w3 * hiring_capacity        # open_roles, headcount growth
           + w4 * pricing_headroom       # no enterprise tier yet
           - w5 * gtm_already_present
           - w6 * enterprise_tier_exists
```

Weights are hand-set, transparent, and live in `ledger/config.yaml` under
`config_version`.

### Slate size and stratification

20 claims/week. Roughly 320 calls by January, ~200 resolved.

Selection is **stratified, not pure top-20 by readiness.** Ranking alone would
over-sample whichever trigger family fires most often — plausibly reaching
January with 140 resolved GTM claims and 11 pricing claims, leaving the research
question unanswerable for every signal but one.

A floor of 3 claims per trigger family per week yields roughly 60 resolved
claims per family by January, which is enough to rank them. Remaining slots go
to the highest-readiness candidates regardless of family.

The cost is real and should be stated: filling a quota sometimes means including
a lower-conviction claim, which slightly worsens the aggregate Brier score. That
is an acceptable trade — a marginally worse score that supports a finding beats
a better score that supports nothing. Claims admitted to fill a quota are
flagged `quota_filled: true` so the effect can be measured rather than assumed.

**Hard ergonomic constraint:** 20 calls must fit in one sitting, so each claim
must render on **one screen** — claim, the three or four numbers that matter,
trailing sparkline, base rate, probability prompt. If `ledger review` requires
scrolling or context-switching, the weekly ritual will be abandoned by October.
This is a design requirement, not a nice-to-have.

### Difficulty filter

Claims whose historical base rate falls outside 15–85% are rejected — a trigger
that resolves true 90% of the time proves nothing and is the first thing a
sharp reader will check for.

Cold start: base rates do not exist in September. Weeks 1–4 run with
`calibration_phase: true`, deliberately spreading thresholds to *learn* the base
rates. The filter is enforced from week 5. Calibration-phase claims remain in
the ledger, flagged, and are reported separately.

---

## 7. Integrity

Three mechanisms, each defeating a specific attack.

### Anti-cherry-picking: the mandatory slate

The model generates the slate; the human answers **every claim on it**. Skipping
is recorded as a skip, visibly. There is no mechanism to decline the hard ones.

### Anti-retroactive-deletion: the weekly manifest

Each Monday, after collection and slate generation, build a manifest containing:

- the commit SHA
- a hash of every snapshot written that week
- a hash of every prediction, open or new
- a **monotonic cumulative count of all predictions ever created**

`ots stamp` the manifest; commit the `.ots` proof.

The cumulative count is the load-bearing field. Stamping individual predictions
alone would permit making 40 calls, publishing the 12 that landed, and having
every proof still verify. To hide a bad call under this scheme the count would
have to shrink — and the prior week's Bitcoin-anchored proof says it did not.

This is what makes private-now-public-later cost nothing.

### Anti-mutation: append-only CI

A prediction file, once written, is immutable except for a single legal
transition:

```
status: open -> resolved | expired
  plus outcome, resolved_at, resolution_evidence
```

Any other diff fails CI. Same pattern as the existing frozen-field check on
`universe.yaml`.

---

## 8. Universe

150 companies, two cohorts, both permanently tagged:

- **~100 systematic** — YC W26/S26 plus Form D rolling, per `SPEC.md`.
  Sector mix ~75% AI / ~25% fintech.
- **~50 target-fund portfolio** — LA-area seed/AI funds.

Fund selection is optimised for **USC alumni overlap** — funds with USC or
Marshall people on the team. This compounds with outreach already underway: a
warm alum introduction combined with *"I've tracked three of your portfolio
companies weekly for four months"* is materially stronger than either alone.

> **Open decision, gated before Sep 7:** the fund list is not yet chosen. It
> must be researched, proposed, and explicitly approved before any entry is
> written, because `slug`, `added` and `cohort` are permanent and CI-enforced —
> this is the one decision in the project that cannot be revised later.

**Consequence to accept:** at ~75% AI in the systematic cohort plus a fund
cohort that will also skew AI, the universe lands near 80/20 overall. Fintech is
descriptive colour, not a cell that supports conclusions. No fintech finding
should be promised.

Separate `cohort` values so every analysis can be reported with and without the
fund cohort. `slug`, `added` and `cohort` are permanent and CI-enforced.

The fund cohort exists because it converts the project into a conversational
opening: *"I've been tracking three of your portfolio companies weekly for four
months."* The selection bias this introduces is disclosed in `RULES.md` and
controlled for by the cohort split.

---

## 9. Signals: intent vs evidence

The collectors in `SPEC.md` measure **intent** — hiring an AE is a company
*saying* it will sell; adding a pricing tier is *preparing* to sell. They
contain no measure of **evidence** that customers exist.

### Revenue is out

No public revenue exists for private seed-stage companies. Estimating sources
(Sacra, Harmonic, PitchBook, Sourcescrub) are four-figure annual costs, produce
modelled guesses, and would put an indefensible number in the ledger.
`RULES.md` records revenue as an explicit named blind spot. This is a better
answer than a soft number.

### Public evidence signals, by value per unit effort

| Signal | Source | Cost | Status |
|---|---|---|---|
| Enterprise-readiness docs (`sso`, `saml`, `scim`, `audit-log`, `soc-2`, `compliance`) | regex over `sitemap.xml` already being fetched | **zero extra requests** | **Weekend 1** |
| Customer logo count + raw names | `/customers`, `/case-studies`, "trusted by" blocks | medium, fragile | Week 2 |
| Case study count | same pages | low, once above exists | Week 2 |
| Community size (Discord/Slack) | public invite pages | medium, spotty | Deferred |
| G2 review count | G2 profile | medium, sparse at seed | Deferred |

The enterprise-docs regex is the highest value-per-effort signal in the project.
Nobody builds SAML for fun; its documentation appearing is the clearest public
tell that a company has landed or is chasing real enterprise customers. It is
also a **level** trigger, so it helps the cold-start problem rather than
worsening it.

### Client quality

Do not classify at collection time. Store raw:

```json
"customer_logos": ["Ramp", "Notion", "Snowflake"],
"customer_logo_count": 14,
"case_study_count": 3
```

Classify tier (enterprise / mid-market / startup) **at scoring time** against a
static reference list. Deterministic, no LLM, and a wrong classifier is re-run
retroactively across all history — the same property `raw_titles` was designed
for, applied to a stronger signal.

`customers.py` ships the robust parts first: page-exists, content hash, and logo
*count* via `<img>`/`alt` heuristics. Logo *names* are best-effort and mark
`status: "partial"`. A noisy name list beats none, and the count alone drives
the triggers.

---

## 10. Research objective

**Question:** which public signal is the single best leading indicator of
commercialization?

A horse race across the trigger families — GTM inflection, pricing move, hiring
threshold, shipping velocity, enterprise motion — ranked by how well each
predicts its own claim resolving. Output is a ranked table with base rates,
resolution rates, and lead times.

This is chosen over a cross-sector comparison because the universe is too
AI-weighted to support one, and over "can a human beat the model" because that
falls out of the head-to-head for free and needs no separate design.

Stratified slate selection (§6) exists to make this question answerable. Without
it, there is no research layer.

---

## 11. The weekly ritual

Abandonment is the single largest risk to the project. The ritual is defended by
a **scheduled task**, with three specifics that matter more than the mechanism:

- **Fires Monday evening, not morning.** Collection runs 09:00 UTC — 2am
  Pacific. A morning notification competes with class and work and gets
  dismissed. A 30-minute reflective task needs an uncontended evening slot.
- **The top 3 claims appear in the notification body.** If seeing what is
  waiting requires opening a terminal, activation energy has been added at
  exactly the point where the habit breaks.
- **`ledger review` shows a streak counter.** Cheap to build, and it surfaces a
  skipped week to the user before it surfaces in the published record.

No calendar block. Two reminders for one task trains the user to ignore both.

---

## 12. Schedule

Today is 2026-08-28. The only immovable date is **2026-09-14**.

| Window | Deliverable |
|---|---|
| Aug 28–31 | `SPEC.md` Weekend 1 verbatim: schema, storage, ATS, GitHub, pricing, changelog, `collect.yml` green, 10 companies. Plus enterprise-docs regex. |
| Sep 1–7 | EDGAR + Apify. `customers.py`. Universe to 150. `RULES.md`. First full run. |
| Sep 8–14 | `ledger/`: schema, resolvers, append-only CI, manifest + OTS. **First predictions logged Sep 14.** |
| Sep 15–21 | `ledger review` CLI. Weekly ritual begins. |
| Sep 22 – Jan 11 | Operate at ~1–2 hrs/week. First resolutions mid-October. Phase B triggers from week 5. |
| November | Model v0.2 from resolved outcomes. Research finding, if one holds. |
| January | ~320 calls, ~200 resolved. Brier, calibration curve, human-vs-model. |

Everything before Sep 14 exists to make Sep 14 possible. Slippage is absorbed by
cutting scope elsewhere, never by moving that date.

---

## 13. Out of scope

No web UI, no database, no LLM calls, no Wayback backfill, no revenue data, no
community/G2 collectors. A public-facing page is a December question, and only
if the record merits showing.

---

## 14. Risks

**Abandonment in week 6.** The realistic failure mode. Mitigated by: GitHub
Actions performing everything except the review; the one-screen CLI constraint;
the Monday-evening scheduled task with claims inline (§9b); the streak counter;
and — most importantly — the model predicting regardless of whether the human
shows up, so the record degrades gracefully rather than stopping, with skipped
weeks honestly visible.

**Coverage collapse.** A structural break (provider API change) silently
degrading the series. Mitigated by the `validate` step failing the run below 50%
coverage, and per-collector coverage reporting.

**A mediocre score.** Acceptable and worth accepting deliberately. A calibrated
forecaster who reports "I was 60% on these, 58% hit, and here is the class of
call I am systematically bad at" is more credible than a suspiciously clean
record, because the first is obviously real.

**Trivial claims inflating both scores.** Mitigated by the difficulty filter
from week 5 and separate reporting of calibration-phase claims.
