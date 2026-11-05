# Phase 0 — US data sources

> **Status:** working doc, filled in during phase 0.
> **Scope:** United States only. India/UK sources are deliberately out of scope (see README §7).
> **Companion:** README §9, Phase 0.

---

## 1. Purpose

Phase 0 is not a data-collection phase. It is a **validity phase**: it answers whether a signal
would have caught a deal *before* the deal happened, at a scale where the work is a spreadsheet and
a few scripts rather than a pipeline.

It answers three questions:

1. **Where does each signal actually come from** — endpoint, licence, cost, refresh rate?
2. **Was it observable at T−12**, or only knowable in hindsight?
3. **Does it separate exits from matched non-exits** by at least 2×?

Sample: **300–500 exits and 2,000 controls.** That is small enough to assemble by hand in two weeks
and large enough to compute hazard ratios with usable confidence intervals.

If phase 0 fails, we have spent two weeks instead of two quarters.

---

## 2. Deliverables

| File | Contents | Acceptance |
| --- | --- | --- |
| `labels.csv` | US technical-startup exits 2019–2025 + matched controls | ≥300 exits, ≥2,000 controls, ≥80% with a source URL |
| `sources.md` | Source scorecard (§10) filled for every candidate | ≥15 sources scored, ≥8 reachable without payment |
| `features.parquet` | Per-company features at T−18 / −12 / −6 / −3 | ≥70% of exits have ≥3 feature families at T−12 |
| `lift.md` | Hazard ratios for H1–H6 with CIs | ≥3 detector families at HR ≥2, CI lower bound >1 |
| `evals/backcast/YYYY-QN.md` | First missed-exit audit (§9.2) | Every sampled missed exit classified; every action owned and dated |
| `decision.md` | Go/no-go, licensed-source shortlist, pack priority | Signed by whoever owns the P&L |

The phase-0 deliverables land in `evals/phase0/`. The backcast series lives in `evals/backcast/` and
continues after phase 0. Nothing here becomes production code.

---

## 3. Label definition

Positive event, strongest to weakest:

1. **`process_started`** — banker retained, LOI signed, or "strategic alternatives" announced.
   Best label; scarce. Use where X deal history or banker pages provide it.
2. **`acquired`** — control change completed. The workhorse label.
3. **`acqui_hire`** — team + IP, no product continuity. Counts, but flag it: the pre-deal signals
   differ (usually a funding failure, not a strategic process).
4. **`shutdown` / `asset_sale`** — negative outcome, useful as a weak positive for the distress
   family only.

Excluded: IPOs, SPAC mergers where the target is the sponsor's vehicle, and deals where the US
company is a subsidiary of a foreign parent.

### `labels.csv` schema

```
company_id, name, domain, cik, sector, business_model, state,
founded_year, last_round_date, last_round_usd, headcount_t12,
event_type, announced_at, closed_at, acquirer_name, acquirer_cik,
consideration_usd, source_url, source_type, label_confidence, is_control, control_match_id
```

### Controls

Four controls per exit, matched on `sector × stage × founding vintage (±2y) × state`. Controls must
still be alive at the exit's T+365 or the comparison is contaminated. Draw controls from the same
universe definition as README §7, not from the whole US company population.

---

## 4. Point-in-time rules

The most common way a backtest lies is by reading today's state into yesterday's decision. Rules:

- Every feature row carries `observed_at`. A feature may only use data with
  `observed_at <= T−x`.
- **Immutable sources** (EDGAR, GH Archive, Wayback, Common Crawl) are trustworthy at any T.
- **Current-state sources** (Crunchbase, company websites, LinkedIn-like data) are backfilled and
  survivorship-biased. Use only their dated event fields (round date, round size), never their
  present-day summary.
- Deleted companies are the majority class in sell-side. If a company's site is gone, reconstruct
  it from Wayback/Common Crawl or drop it from the control set and record the drop rate.

Log every reconstruction decision in `evals/phase0/reconstruction_log.md`. If two people rebuild the
same label and get different features, the feature definition is wrong.

---

## 5. Source inventory

Cost column: `free` = no payment, `key` = free account/key required, `paid` = licence required.

### 5.1 Labels and deals

| Source | Endpoint / access | Cost | History | Join key | Notes |
| --- | --- | --- | --- | --- | --- |
| EDGAR full-text search | `efts.sec.gov/LATEST/search-index` | free | 2001→ | CIK | JSON behind the EDGAR search UI; declare UA |
| EDGAR 8-K Item 2.01 | `sec.gov/cgi-bin/browse-edgar`, full-index | free | 2003→ | CIK | Completion of acquisition. Only **material** deals |
| EDGAR 8-K Items 2.05 / 2.06 | same | free | 2003→ | CIK | Exit/disposal costs, impairments — distress |
| EDGAR S-4, DEFM14A | full-index | free | 1990s→ | CIK | Merger registrations and proxies; "strategic alternatives" language |
| EDGAR full-index files | `sec.gov/Archives/edgar/full-index/` | free | 1993→ | CIK | Bulk enumerations by day/quarter |
| Form D | full-index XML | free | 2008→ | CIK / name | Funding, not exits. Filed ≤15 days after first sale |
| Crunchbase export | API or bulk export | paid | 2010s→ | name + domain | Best private-deal coverage; survivorship-aware; use dated fields only |
| PitchBook | API | paid | 2010s→ | name + domain | Alternative to Crunchbase; usually pick one |
| GDELT 2.0 DOC API | `api.gdeltproject.org/api/v2/doc/doc` | free | 2017→ | name | "acquires", "strategic alternatives"; needs NER + dedupe |
| Google News RSS | `news.google.com/rss/search` | free | varies | name | Fallback recall; thin history |
| Boutique banker transaction pages | 50–200 firm sites | free scrape | varies | name | Slowest, highest precision for `process_started` |

### 5.2 Funding and runway

| Source | Endpoint / access | Cost | History | Join key |
| --- | --- | --- | --- | --- |
| Form D XML | EDGAR | free | 2008→ | CIK |
| Crunchbase rounds | API/export | paid | 2010s→ | domain |
| USPTO assignments | `assignment-api.uspto.gov` | free | 1980s→ | assignee name |

Runway needs round size, round date, headcount, and a burn proxy. Only the first two are clean; the
rest are estimates with wide bands. Store `runway_months` with `runway_uncertainty_months`.

### 5.3 Distress

| Source | Endpoint / access | Cost | History | Join key | Notes |
| --- | --- | --- | --- | --- | --- |
| State WARN notices | 50 separate sites (CA EDD, NY DOL, TX TWC, WA ESD …) | free | varies | name + address | No federal feed. 50 parsers or a paid aggregator |
| Layoffs.fyi | site | free scrape | 2020→ | name | US-heavy; good recall for tech |
| TrueUp | site | free browse | 2022→ | name | Layoff + hiring trend |
| CFPB enforcement | `consumerfinance.gov` actions | free | 2012→ | name | Fintech counterparty and direct |
| OCC / FDIC enforcement | agency sites | free | varies | name | Sponsor-bank consent orders — the fintech shock |
| HHS OCR breach portal | `ocrportal.hhs.gov` | free | 2009→ | name | Healthtech; breaches are existential for small vendors |
| USPTO fee status / trademark §8 | TSDR, Patent Center | free (key) | varies | mark/patent | Lapsed fees and abandoned marks = wind-down |
| Late or missed SEC filings | EDGAR submissions JSON | free | 1993→ | CIK | Public-company distress only |

Note the sequencing problem: distress is usually public **after** a process has started. It is a
confirmation feature, not a prospecting one. Expected hazard ratio is the lowest of the five classes.

### 5.4 Technical decay — the differentiated layer

| Source | Endpoint / access | Cost | History | Join key | Notes |
| --- | --- | --- | --- | --- | --- |
| GitHub REST | `api.github.com` | free (5,000 req/h authed) | — | `github_org` | Releases, contributors, archived, commit cadence |
| **GH Archive** | `data.gharchive.org/{date}-{hour}.json.gz` | free | **2011→** | `github_org` | Immutable; the only clean way to backtest GitHub features |
| npm registry | `registry.npmjs.org` | free | — | package name | Publish cadence, versions |
| npm downloads | `api.npmjs.org/downloads/range/{start}:{end}/{pkg}` | free | 2015→ | package name | Daily downloads |
| PyPI stats | `pypistats.org/api` | free | 2016→ | package name | Download history |
| Docker Hub | `hub.docker.com/v2/repositories/{ns}/{repo}` | free | — | repo | Pull counts |
| Hugging Face | `huggingface.co/api/models?author=` | free | 2022→ | org | Model/dataset downloads |
| ATS job boards | Greenhouse / Lever / Ashby public JSON | free | **none** | `ats_token` | **Snapshot from day one**; backtest via Wayback |
| Wayback CDX | `web.archive.org/cdx/search/cdx` | free | 1996→ | domain | Careers, pricing and changelog diffs |
| Common Crawl | `index.commoncrawl.org` + S3 WARC | free (req-payer egress) | 2008→ | domain | Recovers dead sites; fixes survivorship bias |
| crt.sh | `crt.sh/?q=&output=json` | free | 2015→ | domain | Certificate issuance/expiry history |

ATS boards are the single highest-value free source and the only one with **no history**. Start
daily snapshots during phase 0 even though they cannot help the backtest; they are the data you will
wish you had in six months.

```bash
# public JSON, no auth
curl -s "https://boards-api.greenhouse.io/v1/boards/$TOKEN/jobs"
curl -s "https://api.lever.co/v0/postings/$ORG?mode=json"
curl -s "https://api.ashbyhq.com/posting-api/job-board/$ORG"
```

### 5.5 Process leakage

| Signal | Source | Cost |
| --- | --- | --- |
| CFO / head of corp dev / strategic finance hire at a small company | ATS role titles | free |
| Banker retained | boutique transaction pages, press | free scrape |
| "Strategic alternatives" language | S-4 / DEFM14A, GDELT phrase search | free |
| Data-room vendor mentioned in a job post | ATS text | free |

This class has the highest precision and the lowest volume. Expect a handful of hits per month across
the whole universe; treat every hit as a same-week call, not a score.

### 5.6 Vertical registries (US)

| Vertical | Source | Cost | Join key |
| --- | --- | --- | --- |
| Healthtech | openFDA 510(k) + MAUDE (`api.fda.gov`), ClinicalTrials.gov v2, CMS PFS (`data.cms.gov`) | free (key) | name / NPI |
| Healthtech | HHS OCR breach portal | free | name |
| B2G SaaS | FedRAMP marketplace | free | name |
| Fintech | NMLS Consumer Access | free scrape, **ToS review required** | NMLS ID |
| Hardware | FCC equipment authorization | free | FCC ID |
| Any US federal seller | SAM.gov API, FPDS ATOM feed | free (key) | UEI / CAGE |

NMLS is the one source whose terms must be cleared by counsel before it enters the pipeline. It is
public lookup, but that does not settle whether systematic collection is permitted.

---

## 6. Sources considered and rejected for v1

| Source | Verdict | Reason |
| --- | --- | --- |
| LinkedIn | **Rejected** | ToS breach risk, brittle, unnecessary — ATS boards cover role mix and headcount legally |
| X / Twitter | **Deferred** | Paid, rate-limited, noisy. A manual 200-account founder watchlist beats a pipe |
| Reddit | **Rejected** | Signal-to-noise far too low for exit intent |
| Hacker News | **Narrow use** | Free Algolia API; only "Who is hiring" thread counts for devtools hiring trend |
| Similarweb | **Paid, later** | Traffic trend is useful but the free tiers are too coarse for per-company decisions |
| G2 / review sites | **Deferred** | No API, scraping sits in a ToS grey zone; revisit only if review velocity proves predictive |
| DNS history vendors | **Paid, later** | Certificate transparency covers most of the need in v1 |

The rule: if a source requires violating a contract to read it, it does not enter the pipeline. A
lead that cannot be defended to a regulated acquirer is not an asset.

---

## 7. Join keys and entity resolution

One canonical `company_id` per company, keyed by **registrable domain** first. Every source attaches
through `company_external_id(company_id, scheme, value)` from README §4.

| Scheme | Source | Collision risk |
| --- | --- | --- |
| `domain` | company site | primary key; brands on subdomains need a merge rule |
| `cik` | EDGAR | safe; one CIK can hold many brands |
| `github_org` | GitHub | safe |
| `package` | npm/PyPI | scoped names are safe; unscoped collide |
| `nmls_id` | NMLS | safe |
| `uei` / `cage` | SAM.gov | safe |
| `ats_token` | Greenhouse/Lever/Ashby | safe, but tokens change on ATS migration |
| `name_state` | WARN, state registries | high collision risk; always pair with city + domain |

Name-only matching is the biggest source of silent error in a label set. For phase 0, every label
needs a domain or a CIK; anything that resolves on name alone is marked `label_confidence: low`.

US entity resolution shortcut: Delaware is the incorporation home for most VC-backed companies but
its registry is paid and fragmented. Do not build on it. Domain + CIK + GitHub org is enough.

---

## 8. Legal and compliance (US)

- **SEC fair access:** max 10 requests/second, `User-Agent` with a real contact address. Exceeding it
  gets the IP blocked and is the one hard technical rule in EDGAR.
- **Public does not mean free to collect.** `hiQ v. LinkedIn` narrowed CFAA exposure for public
  pages but contract and ToS claims survived. Treat ToS as binding.
- **State privacy law.** CPRA removed the B2B and employee exemptions on 2023-01-01, so business
  contact data is personal information in California. Founder-level intent inference is in scope;
  keep the lawful-basis note, deletion path and suppression list even for US-only data.
- **Enforcement and breach data** are public records, but republishing a claim that a named company
  is distressed is a defamation surface. Internal scoring only; never in a customer-facing field.
- **Licensed feeds** (Crunchbase, PitchBook, secondary-market data) carry redistribution limits. Do
  not blend them into a dataset you have not cleared.
- **Retention:** raw HTML 90 days, extracted facts indefinitely, personal fields on review.

---

## 9. Backtest protocol

For each exit and control, compute every feature at **T−18, T−12, T−6, T−3**, where T is
`announced_at`.

```
HR(detector) = P(detector active at T−12 | exit) / P(detector active at T−12 | control)
```

- Report Wilson 95% CIs. Do not report a detector with fewer than 10 positive cases.
- Report median lead time as the first T window at which the detector separates at HR ≥ 2.
- Recompute base rate over the same window and geography. If the base rate moves, the lift number
  moves with it — quote both.
- Controls are matched on pre-T characteristics and are **not** conditioned on surviving past T;
  conditioning on T+365 selects on the outcome and biases every ratio. See
  [pre-code-gaps.md](pre-code-gaps.md) §A1.

Hypotheses under test: H1 bridge round, H2 release cadence, H3 founder/director departure,
H4 corp-dev or CFO hire, H5 runway <9 months, H6 two or more decay signals simultaneously
(README §5.5).

### 9.1 Recall, not just lift

Two rates, because a rule that never fired and a rule that fired but was not served are different
failures:

```
missed_exit_rate_fired(w)  = in-universe exits with no detector firing at T−w
                             / in-universe exits with a reconstructable feature family at T−w
missed_exit_rate_served(w) = in-universe exits that would not have entered the shortlist at T−w
                             / in-universe exits with a reconstructable feature family at T−w
coverage(w)                = in-universe exits with a reconstructable feature family
                             / all in-universe exits
```

- `fired` is pre-threshold; `served` applies the ranking and slot budget in force at T−w. Report
  both. A ranker or threshold change must not masquerade as a detector change.
- `fired` counts any detector that met its own threshold, right or wrong. It measures event-level
  recall, not attribution; per-detector precision belongs in `lift.md`.
- The denominator is in-universe exits with at least one reconstructable feature family. Report
  `coverage` beside the rate so the excluded mass is visible rather than hidden inside the metric.
- Exits outside the served universe are reported separately. No detector can fire on a company that
  was never collected, so they are a universe decision, not a detector defect.
- Report at T−12 and T−6. A family with HR ≥ 2 can still miss most exits if it is rare; lift and
  coverage are separate numbers and both belong in `lift.md`.
- Evaluate with the detector set that was live at T−w, and record `detector_version` plus
  `mode = as_live | retrospective`. Applying today's rules to old exits is a useful R&D test, but it
  is not the as-live rate and must not be blended into it.
- README §10's target of ≤ 0.40 applies to `missed_exit_rate_served@T−12`. Phase 0 has no ranker, so
  it gates on `fired` and sets the baseline in `decision.md`. Do not tune either after the result.

Miss classes, once an exit is in-universe, observable and missed:

| Class | What it means | Action |
| --- | --- | --- |
| **Uncaptured** | no source covered the precursor | source registry; `source_discovery` |
| **Captured, undetected** | the feature exists, no rule fired | hypothesis backlog; backtest before promotion |
| **Detected, not served** | the rule fired, the ranker or slot budget excluded it | calibration, not new data |
| **No observable precursor** | no public trace at T−w | no detector fix; report as the achievable floor |

Two exclusions, reported but not counted as detector misses:

| Exclusion | Meaning | Action |
| --- | --- | --- |
| **Outside universe** | the company was never in the served universe at T−w | universe decision |
| **Archive too young** | T−w predates the archive | revisit next quarter; no source verdict |

Do not remove "no observable precursor" exits from the rate after classifying them — that is
post-hoc selection. Report their share so the floor is explicit.

### 9.2 Quarterly backcast

Run it every quarter, starting at the end of Phase 0. The ritual classifies misses; the rates are
computed on the full mature cohort, never on the sample.

1. Cohort: exits whose outcome is final (closed or terminated) with `announced_at` 6–15 months before
   the audit. T−6 must be inside the archive. Audit T−12 where the archive reaches it; otherwise
   record `archive too young` for that window rather than calling it uncaptured. Pending deals are
   excluded until they close or terminate.
2. Compute both rates and `coverage` on every in-universe exit in the cohort. No sampling for the
   numbers.
3. For classification, take every missed exit if there are ≤30; otherwise a stratified sample of 30
   by pack × size band, plus every exit above the 90th percentile by last round size (or
   consideration where disclosed).
4. Blind classification. Two coders independently label each sampled exit from the T−w archive alone,
   before seeing the why-now narrative. A separate analyst writes that narrative from post-close
   sources — 8-K, acquirer statement, press, founder post. Adjudicate disagreements and record the
   adjudication. Report raw agreement alongside Cohen's κ, because κ collapses when one class
   dominates the sample. A class that cannot be agreed is `unknown`, not forced.
5. A class must rest on a dated, rule-checkable observation in the archive. "The narrative makes
   sense of this signal" is not evidence; hindsight will manufacture a precursor out of ordinary
   noise.
6. Reconcile the cohort against deals announced since the last audit, so a missing label is not
   counted as a missed detector.
7. Pair the audit with the quarter's precision@50. A lower miss rate bought by firing more rules is
   not progress; precision is the other half of the trade.
8. Publish `evals/backcast/YYYY-QN.md` with the rates, `coverage`, the class counts, the three largest
   classes, agreement and κ. Report class shares reweighted back to the cohort: the stratified sample
   and the 90th-percentile round overstate their strata if raw counts are quoted. Every action gets
   an owner and a due date; last quarter's actions are reviewed before new ones are opened.

Report the rate; do not chase it to zero. Some exits are genuinely unforecastable, and a team that
optimises this metric will start fitting noise.

---

## 10. Source scorecard (fill during week 1)

Working sheet: [`evals/phase0/sources.md`](../evals/phase0/sources.md). Score each source on what it
actually delivered, not what the docs claim.

| Source | Labels covered | Fields usable | Refresh | History depth | ToS risk | Cost | Verdict |
| --- | --- | --- | --- | --- | --- | --- | --- |
| EDGAR 8-K 2.01 | | | | | low | free | |
| EDGAR full-text | | | | | low | free | |
| Form D | | | | | low | free | |
| Crunchbase | | | | | medium | paid | |
| GDELT | | | | | low | free | |
| WARN (per state) | | | | | low | free | |
| Layoffs.fyi | | | | | medium | free | |
| GitHub API | | | | | low | free | |
| GH Archive | | | | | low | free | |
| npm / PyPI | | | | | low | free | |
| ATS boards | | | | | low | free | |
| Wayback CDX | | | | | low | free | |
| Common Crawl | | | | | medium | free | |
| USPTO | | | | | low | free | |
| openFDA / CT.gov | | | | | low | free | |
| FedRAMP | | | | | low | free | |
| NMLS | | | | | **high — counsel** | free | |

"Labels covered" = share of the 300–500 exits for which the source yielded an independently
verifiable data point. A source that scores below 10% is dropped from the phase-1 build regardless
of how interesting it looks.

---

## 11. Week plan

**Week 1 — free sources, no payments**

| Day | Work |
| --- | --- |
| 1 | Universe definition; start ATS + careers-page snapshots; label schema frozen |
| 2 | Pull EDGAR 8-K 2.01 / S-4 / Form D for the label window; seed `labels.csv` |
| 3 | Fill label gaps from GDELT + press; add Crunchbase trial rows for the same 50 deals to measure coverage delta |
| 4 | GitHub + GH Archive + npm/PyPI collectors for feature reconstruction; Wayback CDX for careers/pricing |
| 5 | Build controls; first pass at H2 and H5 |

**Week 2 — keys, licences, decision**

| Day | Work |
| --- | --- |
| 6 | USPTO, openFDA/ClinicalTrials, FedRAMP; WARN for the 10 largest states |
| 7 | Compute `features.parquet` at all four windows; log reconstruction decisions |
| 8 | Lift table for H1–H6; first `missed_exit_rate_fired@T−12` audit with misses classified; quantify survivorship and label bias |
| 9 | Source scorecard; licence shortlist with indicative pricing |
| 10 | `decision.md`: go/no-go, pack order, phase-1 source list |

---

## 12. Go / no-go

Proceed to phase 1 only if all four hold:

- ≥300 exits and ≥2,000 controls assembled with source URLs.
- ≥3 detector families at HR ≥ 2 with a CI lower bound above 1.
- Median lead time ≥9 months for at least two families.
- ≥70% of exits have ≥3 feature families reconstructable at T−12.

If two or more fail: rewrite the thesis before writing a crawler. A pipeline built on a signal that
does not exist is the most expensive possible outcome, and phase 0 is the only cheap place to find
that out.

Also report `missed_exit_rate_fired@T−12`, `coverage@T−12` and the miss-class breakdown from §9.1
(uncaptured, captured-but-undetected, detected-but-not-served, no observable precursor). It does not
block phase 1 on its own — phase 1 adds sources that may close uncaptured misses — but a rate above
the target set
in `decision.md` makes the detector set incomplete by definition, and the largest miss class becomes a
phase-1 work item rather than a later surprise.

---

## 13. Open questions

- [ ] Crunchbase or PitchBook — one licence, not both. Decide on measured label coverage, not on demo quality.
- [ ] Does the acqui-hire label warrant its own detector set? Pre-deal signals look like funding failure, not strategic process.
- [ ] Who owns the 50 state WARN parsers — build in-house or license an aggregator?
- [ ] Is NMLS collection permissible under its terms? Counsel answer needed before fintech pack work starts.
- [ ] Do we extend the label window back to 2015 to get a second economic cycle, or is 2019–2025 enough?
