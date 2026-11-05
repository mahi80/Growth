# Pre-code review — what the plan still doesn't answer

> **Status:** decision document. Review of README + `docs/` as of commit `d7b1434`.
> **Purpose:** the questions that are cheap to settle now and expensive to discover in month three.
> Not a re-litigation of sources — §5.2 and §5.7 already over-deliver there.

The plan is strong on *where signal lives* and *how the pipeline is shaped*. It is thin on
**measurement validity**, **what we are legally allowed to sell**, and **who absorbs the output**.
Four items below can invalidate Phase 0 or the product after it; the rest are decisions that cost a
day now and a rewrite later.

Priority: **P0 = before Phase 0 labels**, **P1 = before Phase 1 code**, **P2 = before serving**.

---

## A. Measurement validity

### A1. The control group is selected on its own future — P0

`docs/phase0-us-data-sources.md` §3 and §9 require controls to be **alive at the exit's T+365**.
That conditions control selection on a post-baseline outcome. It is selection bias (a cousin of
immortal-time bias), and it removes exactly the companies the distress family is supposed to
separate from: the ones that died instead of selling.

A company that shuts down is not a clean negative, and it is not a positive either — it is a
**competing event**. The fix is not a better filter; it is the right estimand:

- Draw controls from the **point-in-time universe active at T**, matched on covariates measured at T.
- Follow both cohorts forward to a common horizon; treat `shutdown`, `ipo`, `still_private` as
  outcome states, not as exclusions.
- Report **cause-specific hazards** (sale vs shutdown) and, as a sensitivity check, a
  Fine–Gray subdistribution. If the sale signal survives both, it is real.
- Keep a small "alive at T+365" subset only as a secondary, clearly-labelled comparison.

Also: matching on `stage` and `sector` needs those fields **as of T**, not as of today. From
current-state sources they are backfilled (§4 already warns about this) — so stage must come from
the dated round fields, or the match is made on future information.

### A2. What §9 computes is a risk ratio, not a hazard ratio — P0

```
HR(detector) = P(active at T−12 | exit) / P(active at T−12 | control)
```

That is a **risk ratio at a landmark**, with no time dimension. It is a perfectly good statistic —
call it one. If the plan wants hazard ratios, fit a Cox model with time-varying detector state and
report the coefficient. Do not mix the labels, because the Phase 3 ranker will inherit whichever
number the decision doc blesses.

Three additions that matter more than the estimator:

- **Cluster by company.** A company contributes four landmark rows; those rows are not independent.
  Cluster-robust standard errors, or one row per company with the earliest usable landmark.
- **Multiple testing.** §5.7.4 adds five sources, the Phase 0 doc lists ~25 more, and six hypotheses
  are tested at four horizons. At that count, "three families at HR ≥ 2" is a coin-flip away from
  happening by chance. Pre-register the primary set (H1–H6) and apply Benjamini–Hochberg to the
  rest; report the q-value next to the CI.
- **Time fixed effects.** 2019–2025 contains a funding peak, an exit drought and a rate shock. Exit
  base rates by year differ by multiples. Train on ≤2022 and test 2023–2025, or at minimum stratify
  by year; otherwise the model learns "2021" as a feature.

### A3. Lead time, as defined, is circular — P0

§9 defines median lead time as "the first T window at which the detector separates at HR ≥ 2".
That is a property of the backtest, not of the detector: with four windows and dozens of detectors,
the earliest separating window is selected *because* it separates.

Define it in the data, not in the result:

- A detector's lead time is the time from the **start of its earliest continuously-active run** to
  `announced_at`, conditional on the detector still being active at the last landmark before T.
- Report the **distribution** (median, IQR, 10th percentile), not the median alone. The 10th
  percentile is what the rep experiences.
- A detector that fires at T−18 and is off by T−12 has a lead time of zero for actionability. Do
  not let it set the headline.

### A4. Verification measures plausibility, not outcome — P1

§10's binding metric is `precision@50 on analyst-verified accounts ≥ 20%`. An analyst confirming
that a company "looks like it might sell" is a **proxy**. It is not the label. A shortlist can score
80% analyst-verified and 0% sold.

Add the only metric that cannot be gamed:

- **Cohort outcome rate.** For each weekly cohort, the share that announces a process or a sale
  within 6 / 12 / 18 months, against a matched control cohort drawn at the same date.
- Track it with a lag, keep the cohorts immutable, and make it the number the business is steered by.
- `precision@50` stays as an operational QA metric — it catches broken evidence, not a broken thesis.

The current §10 metrics also cannot distinguish a model that ranks well from a universe that was
pre-filtered to likely sellers. Cohort outcome rates solve both.

### A5. "300 accounts a week" does not survive the arithmetic — P1

Every number in the plan is plausible on its own; together they are not.

- Universe 50k–150k, annual exit base rate 1–3% (§10), so ~1–4.5k exits a year.
- 300 accounts/week is 15,600 a year — more than the entire exit population at the top of that range.
- 50k universe × 4% (the `base_rate_12m` in the pack YAML) = 2k exits. 15,600 accounts is 8× the
  whole event population, i.e. the list is a **ranking surface**, not a prediction set. That is fine,
  but it must be stated, because "300 sellers a week" is not what the arithmetic says.
- The base rate itself is contradictory: README §10 says 1–3%, the `fintech_lending` pack pins
  `base_rate_12m: 0.04`, and §6.4 gives no provenance for either. One number, sourced from Phase 0,
  per pack, with a CI.

Then the real constraint appears: **absorption**. A boutique advisor can pursue a handful of new
mandates a month, not 1,300 a year. So:

- Define the weekly output as a **global** ranked list that is allocated across customers
  (by sector, geography or exclusivity), each customer seeing ~10–30 names.
- Derive list size from the customer's capacity and the measured precision-at-capacity curve, not
  the other way round.
- Publish precision@K for K ∈ {25, 50, 100, 300} in Phase 1. Without that curve, "how many accounts"
  is a design preference rather than a product decision.

### A6. Nobody measures the denominator — P1

Only shortlisted companies get verified, so the verification log can estimate precision but never
the base rate, and the ranker trains on its own selection.

- Verify a **random sample of 100 non-shortlisted companies per month** on the same rubric. That is
  the base-rate estimate and the only honest calibration set.
- Freeze a stratified eval set before Phase 3 and never train on it.
- Expect the ranker to overfit 50 labels/week quickly; active learning should add *uncertain* and
  *control* rows, not just more positives.

### A7. Label rights are unresolved and block the first purchase — P0

`docs/phase0-us-data-sources.md` §3 says to use "X deal history". Two questions have to be answered
in writing before a single label is loaded:

1. **Who owns that history?** If it came from a prior employer or a platform's transaction log, X
   may not have the right to use it for a new product, let alone to train a model. A licence or an
   assignment is needed, not an assumption.
2. **What do the paid feeds allow?** Crunchbase/PitchBook terms commonly restrict building a
   derivative dataset or a competing product. "We will only use dated fields" is a data-minimisation
   choice, not a licence grant. Counsel reads the contract before the export is bought.

Fallback that needs no licence: banker transaction pages + EDGAR + press, at lower coverage and
higher assembly cost. Phase 0 should measure that coverage (the doc's "Crunchbase trial vs free
sources on the same 50 deals" is exactly the right test) and decide from the delta.

---

## B. What we are allowed to sell

### B1. Process-leak detection can create MNPI exposure — P0

A certificate for `dataroom.<company>.com`, a banker's transactions page, a "strategic alternatives"
sentence: each is public, but together they reveal an **unannounced process**. Selling that inference
is not the same as selling a scraped page. Risks:

- The customer may be a public company (or advise one) and trade or act on it — insider-trading and
  Reg FD exposure that starts with our brief.
- A banker's engagement is confidential; a product that advertises "we spotted your process" invites
  breach-of-confidence claims and destroys the relationship the product needs.
- The claim can be wrong, and a wrong "this company is in a process" is defamatory in a way a wrong
  score is not.

Policy to adopt before the first process-leak lead ships:

- Inference only from public artifacts; never solicit or accept insider tips.
- Never state "in process" about a named company. State the observable fact and its date, and let the
  analyst draw the conclusion.
- No outreach script that implies knowledge the sender should not have.
- Legal review of the process-leak class before Phase 2, not after.

### B2. The value decays with distribution — P1

The pitch is "before the banker". If twenty customers receive the same account the same week, the
first-call advantage is gone and the product is a lead list with a decaying edge.

- Sell by **exclusive territory or sector**, or by allocation, and price exclusivity.
- Measure time-to-first-contact per account; if customers cannot absorb the list, exclusivity is the
  product.
- Decide the X-advises-versus-X-sells-leads fork (§13) before Phase 3. If X runs mandates, it is
  competing with its own customers for the same companies.

### B3. The retention policy contradicts auditability — P2

§8 keeps raw HTML for 90 days but promises every served lead can reproduce
`field → value → source URL → timestamp → method` indefinitely. After 90 days, a URL that has
changed or died cannot be re-verified, so the audit promise quietly expires.

- Store the **evidence span** (quote + offsets + content hash) as the durable artifact, with a
  compressed excerpt, and treat the raw page as cache.
- Keep snapshots for leads that were actually served, under a separate retention rule.
- Test it: pick a lead served a year ago and reproduce the brief from the database alone.

### B4. The product has never been shown to a buyer — P0

There is no customer step anywhere in Phase 0 or 1. The nearest thing is "an analyst checks 100 of
them". The plan can therefore pass its own gate while being unsellable.

During Phase 0, alongside the backtest:

- 10 structured interviews with boutique M&A advisors / corp-dev leads. Show 20 mock briefs, ask what
  they would do differently on Monday, and what they pay for today.
- 1 written pilot commitment (even unpaid) with a named firm.
- Write down the price hypothesis and the unit economics: cost per verified lead × leads per mandate
  versus a mandate fee. §11 is a cost model with no revenue side.

The output of this is a one-page `docs/customer-evidence.md`, not a deck.

---

## C. Schema and reproducibility

### C1. The database cannot express "as of T" — P0

The model has `observed_at` on facts and events, but:

- `company.status`, `stage`, `headcount`, `universe_tier` are **overwritten**. You cannot reconstruct
  which companies were active, in which tier, on 2023-06-01 — which is the denominator of every
  backtest and the definition of the control pool.
- `event.observed_at` is when we saw it; the deal happened earlier. There is no `occurred_at`.
- `dependency` has `discovered_at` but no validity window; a partner relationship that ends in 2024
  is still an edge in 2026.

Minimum fix before Phase 1: effective-dated (`valid_from`, `valid_to`) rows for company attributes,
dependencies and universe membership, plus `occurred_at` on events. Every feature query is then
`where valid_from <= T and (valid_to is null or valid_to > T)` rather than a join on today's state.
`docs/ontology-and-rdf.md` already does this for the RDF export; the operational tables lag it.

### C2. No frozen dictionaries means no reproducible lift table — P0

`signal.type` is free text; detector logic lives in YAML and Python; `lead_time_days` is "assumed,
then fitted" and stored per row. Two analysts will produce two lift tables.

Before Phase 0's first measurement, freeze:

- **Event dictionary**: event kind, fields, date semantics, source of record.
- **Signal dictionary**: id, class, template, inputs, polarity, units, minimum evidence, suppression
  rules, owner. `signal.type` becomes an enum referencing it.
- **Detector and pack versions** on every emitted signal, and a `feature_snapshot_hash` on every
  score, so a score can be recomputed from stored inputs.
- Unit rules: currency, dates, headcount definition (employees vs FTEs vs W-2s), "engineer".

A lift table is a scientific claim; without this it is a screenshot.

### C3. Entity resolution has a target but no gold set — P1

"Duplicate rate < 2%" is unmeasurable without a labelled sample. Phase 1's exit criterion covers
field accuracy and not ER precision/recall, even though ER errors silently move companies between
exit and control.

- Build a 500-pair gold set (200 merges, 200 non-merges, 100 ambiguous) from the Phase 0 universe.
- Report precision/recall per merge rule, not one blended number.
- Treat `ats_token` migration and `github_org` transfer as first-class merge events, not edge cases.

### C4. The universe is still undecided and partly unobservable — P0

§7 says "India first"; the Phase 0 doc says US-only; the lens changed mid-plan. This is the single
largest scope fork: it changes entity resolution, the source registry, the label set and the 10-week
schedule. Pick one before Phase 0 labels are assembled, and make the other an explicit non-goal.

Separately, "10–200 engineers" is not observable at the scale claimed without LinkedIn, which §8
forbids. Replace it with an operational rule and a stated coverage estimate:

- Inclusion requires a resolvable domain **and** at least one of: ATS board, GitHub org, EIN/CIK,
  or a dated funding event.
- Headcount is a band from ATS role mix + LCA + GitHub contributors, with an explicit `unknown`
  bucket. Companies with no evidence are **out of universe**, not zero.
- Report the share of the seed list that fails inclusion — that number is the coverage bias.

Seeding matters too: where do the first 5k domains come from? GitHub orgs, ATS token lists, funding
announcements, state registries — each has a different bias. Write the seed list provenance into
`docs/universe.md`; it will be the first thing a customer asks about.

---

## D. Operations that get expensive later

### D1. No collector contract — P1

A source registry exists; a collector specification does not. Every collector needs, before the first
one is written: source id, endpoint, auth, rate and daily budget, fixture, extracted schema, PIT
snapshot format, failure semantics, expected yield, owner, and a kill switch. This is what keeps 50k
companies maintainable with a small team.

### D2. Serving surface is undefined — P1

`serve/` is a box labelled "API, briefs, verification queue". Decide the first surface now, because
it determines auth, audit and the brief renderer:

- v1: one weekly brief per customer, delivered as email + a CSV/Sheet and pushed to the CRM
  (Affinity / Salesforce / HubSpot). No web app, no public API.
- Every served lead must render its evidence spans and its scoring date. That is the same data the
  audit test in B3 uses.

### D3. Secrets, PII and cost caps — P1

Paid feeds, proxies and the model gateway all need keys; none of this is in the plan. Minimum: keys
outside the repo, no PII in logs, per-source daily cost caps wired to the kill switch, and an
alert when `last_nonzero_at` goes stale. One afternoon, and it prevents the worst incident class.

### D4. Phase 0's week one is four to six weeks of work — P0

400 labels researched to a source URL is the critical path: at ~20 minutes each that is ~130 hours
before controls, features across four windows, archive reconstruction and the lift table. The gate
is good; the schedule is not.

Split it:

- **Days 1–3: feasibility spike.** 50 exits, three features, archive-only reconstruction. Answers the
  only question that can kill the design: *how much of the signal is actually reconstructable at
  T−12?* If under ~60%, the feature set changes before the label work scales.
- **Then 4–6 weeks: the full backtest**, with the label window and hypotheses frozen.
- Phase 1 does not start until `decision.md` is signed. The current week numbering implies Phase 1
  begins while Phase 0 is still producing features.

Counterparty monitoring also has no budget line: §6 propagates from hundreds of banks, EHRs,
marketplaces and agencies. That is a second crawler with its own registry, cadence and failure modes.

---

## E. Sequencing that is still wrong

1. **Phase 0 labels before buyer evidence.** Both are cheap. Do the 10 interviews in the same week;
   they may change what counts as a good lead (a pitch list for a banker, not a "seller" feed).
2. **Ranker objective.** `propensity = P(process within 12 months)` optimizes for exits, not for
   revenue. Two more terms are needed before Phase 3: expected fee (deal size band) and **capture**
   (probability we or our customer is the advisor). Ranking on P(exit) alone surfaces companies that
   will sell through someone else.
3. **Pack base rates before packs.** `calibration: { base_rate_12m: 0.04 }` is a guess inside a
   config file. Replace with Phase 0 measurements per pack, or a prior with an explicit interval.
4. **Process-start labels have a selection bias.** Deals with visible banker pages skew larger and
   advisor-represented; the label set will overstate precision for the smallest targets. Report label
   coverage by deal size.
5. **"Three feature families" needs a diversity rule.** Three website-derived features are one
   source. Require at least one family independent of the company's own site (registry, filing, code,
   contract clock).

---

## F. The week-one checklist

| # | Action | Owner | Output |
| --- | --- | --- | --- |
| 1 | Label-rights memo: prior history + paid feeds | founder + counsel | `docs/label-rights.md` |
| 2 | MNPI / process-leak policy note | counsel | section in `docs/process-leak-policy.md` |
| 3 | Rewrite control design + analysis plan (A1–A3) | quant | updated Phase 0 §3/§9 |
| 4 | 3-day archive feasibility on 50 exits | data | `evals/phase0/feasibility.md` |
| 5 | 10 buyer interviews + 20 mock briefs | founder | `docs/customer-evidence.md` |
| 6 | Freeze event / signal / label dictionaries | eng | `docs/dictionaries.md` |
| 7 | Verification rubric + monthly control sample (A4, A6) | eng | `docs/verification-protocol.md` |
| 8 | Pick the country and the serving surface | founder | decision in §13 |

Nothing in this list requires writing production code, and every item either shrinks the Phase 0
scope or prevents a rewrite.
