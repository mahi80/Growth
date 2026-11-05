# People, headcount and attrition — capture design for `talent_exodus`

> **Status:** proposal, pre-code. Companion to README §5.2, §5.7 and §6.2.
> **Scope:** US v1. This is about *observing* movement that already happened, never about scoring
> a person's likelihood of leaving.

---

## 1. A departure is an absence, and absences decay into three weaker observations

You never observe "an employee left". You observe one of:

1. **A roster slot is empty** — a name disappears from an authoritative list (state filing, licence
   roster, plan filing, patent assignment).
2. **A page changed** — team, leadership or about page no longer lists the role.
3. **A trace stopped** — commits, releases, package publishes, conference bios, bylines.

Each has a different failure mode, and conflating them is the main way a "people" signal becomes
noise:

| Class | Example | What a disappearance means | Failure mode |
| --- | --- | --- | --- |
| **A. Authoritative roster** | state annual report, NMLS sponsorship, Form 5500, LCA | the role is genuinely vacant | company does not file / file is stale; name variants |
| **B. Curated page** | team page, leadership page, about page | the role *appears* vacant | redesign, reordering, lazy load, A/B test → false "mass exit" |
| **C. Incidental trace** | commit authors, releases, package maintainers, speaker bios | the person is inactive on that surface | holidays, private repos, new maintainer with different login |

Rule: **only class A can emit a `departure` event.** Class B emits `vacancy` with a confidence and
requires corroboration. Class C emits `inactivity` and is a feature, never an event.

This single rule prevents the most common false positive in the category: a team-page redesign read
as the entire leadership team quitting.

---

## 2. Why a feed of single departures is not a signal

- **Normal attrition swamps it.** Technical attrition runs in the mid-teens to ~20% a year and varies
  by role (sales higher than engineering). A 60-person company loses ~10 people a year without
  anything being wrong. One departure carries almost no information.
- **Direction of causality differs by level.** Founder/CFO/CTO departures tend to *precede* a sale
  (the board has decided, or the executive sees the runway). Individual-contributor churn often
  *follows* a process — retention packages, uncertainty, rumour. IC attrition is a confirmation
  feature; leadership vacancy is a prospecting feature.
- **Gross is not net.** A company losing three engineers while hiring five is growing. The unit of
  signal is a **role-class state over time**, not an event feed.
- **Post-announcement churn is a different phenomenon.** Once a process is public, departures are an
  outcome, not a predictor. Censor them using `process_started` dates before measuring lift.

Consequence for the schema: model `role_vacancy(company, role_class, valid_from, valid_to)` and
`headcount_series(company, source, observed_at)`, not a stream of "person left" rows.

---

## 3. Four capture channels

| Channel | What you get | Coverage | Lag | Cost | Legal |
| --- | --- | --- | --- | --- | --- |
| **Authoritative filings** | officer changes, licensed-role counts, plan participants, work-visa roles | low–medium, state/vertical dependent | 1–12 months | free | public records |
| **Curated pages** | team/leadership names, open roles | medium; biased to companies that market themselves | 0–7 days | free | ToS + robots |
| **Incidental traces** | maintainer activity, releases, bylines, speaker bios | narrow but deep in devtools | 0–7 days | free | public APIs |
| **Licensed workforce data** | employment spells with start/end dates, title, seniority | highest, 60–80% of white-collar | 1–3 months | per-record, material | contract; derived-signal rights |

The fourth channel is the only one that directly answers "did this person leave", and it is the one
line item worth pricing early. It is also the lawful substitute for the source everyone reaches for
first — LinkedIn (§8). The other three are free and, combined, cover the executive layer well enough
to start.

---

## 4. Sources worth adding beyond §5.7

### 4.1 Authoritative rosters (class A)

| Source | What it yields | Cadence | History | Join key |
| --- | --- | --- | --- | --- |
| State annual reports (CA Statement of Information, FL, NV, WA, …) | officer/director names and changes | annual per company | varies by state, often 5–10y online | legal name + state |
| Form 5500 (DOL EFAST2) | 401(k) participant counts, plan termination, final return | annual | 2009→ | EIN / plan sponsor |
| DOL LCA / PERM disclosure | employer-visible job titles, wage levels, worksites | quarterly | late 2000s→ | EIN / FEIN |
| NMLS Consumer Access | mortgage-loan-originator roster and sponsoring lender | live | limited | NMLS ID |
| FINRA BrokerCheck | registered reps and their employment history | live | long | CRD |
| State contractor licence (e.g. CSLB) | licence qualifier disassociation | live | long | licence no. |
| State insurance producer appointments | producer–carrier appointments ending | live | varies | NPN |
| PatentsView (USPTO) | inventor disambiguation, assignee, filing dates | quarterly bulk | 1976→ | inventor ID / assignee |
| USPTO assignment records | IP assigned away, security interests | weekly | 1980s→ | assignee |

The strongest of these for a technical universe are **annual-report officer changes**, **Form 5500
participant counts** and **patent-inventor movement**. Each is a genuine departure/headcount event,
each is free, and each is historical — the rare combination that makes a Phase 0 backtest possible.

### 4.2 Headcount and hiring proxies (not departures)

| Source | What it yields | Cadence | History | Join key |
| --- | --- | --- | --- | --- |
| ATS boards (Greenhouse / Lever / Ashby JSON) | open roles, department, office, posting age | daily | **none** | `ats_token` |
| Team / leadership / about page | named roles, claimed headcount | weekly | Wayback only | domain |
| USCIS H-1B Employer Data Hub | approvals per employer per fiscal year | annual | FY2009→ | employer name |
| HN "Who is hiring" (Algolia API) | self-reported hiring and role counts | monthly | 2011→ | company name |
| App store / Play listing | last update, release cadence | weekly | partial | app ID |
| GitHub org + GH Archive | member events, commit/release activity, bus factor | daily | 2011→ | org / login |
| npm / PyPI / Docker Hub / Hugging Face | publish cadence, maintainers, downloads | daily | partial | package / org |
| GitHub Sponsors / Open Collective | funding level for OSS projects | monthly | short | org |

ATS boards are the highest-value *hiring* surface and the cheapest, but they say nothing about
departures — a role disappearing can mean filled, frozen or cancelled. Treat posting lifecycle as
demand, not headcount.

### 4.3 Physical and commercial footprints (manual, high precision)

| Source | What it yields | Use |
| --- | --- | --- |
| Commercial sublease listings (LoopNet, local brokers) | office space being sublet | contraction, often before layoffs |
| Equipment auction listings (Ritchie Bros, BidSpotter) | lab/office equipment being liquidated | wind-down, hardware especially |
| Import/export manifests (Panjiva, ImportGenius — paid) | shipment volume collapse | hardware/consumer |
| Conference, meetup and podcast bios (Wayback) | who was "at company X" in year Y | historical exec/eng roster |
| Local business licences and permits | location closures | retail/consumer |

These are not crawler targets. They belong in an analyst queue with a source link, one row per
observation. Their precision is high enough to justify the manual cost; their volume is low enough
that it stays manual.

---

## 5. Detection rules

```
vacancy(c, r, t)      = roster(c, r, t0) and not roster(c, r, t) and (t - t0) >= 30d
backfill(c, r, t)     = posting(c, r) within 60 days of vacancy
critical(r)           = r in {ceo, cfo, cto, cro, head_of_sales, corp_dev}
net_flow(c, q)        = departures(c, q) - arrivals(c, q)          # licensed data
z_flow(c, q)          = (net_flow - peer_median) / peer_mad         # peers: size × stage × role
roster_drop(c, lic)   = count(lic, now) / max(count(lic, trailing 12m)) < 0.7
payroll_drop(c)       = participants_5500(y) / participants_5500(y-1) < 0.8
maintainer_gap(c)     = days_since_last_commit(top2) > 2 × baseline
```

Fire the class only when **two independent rules** hold within 180 days, mirroring the two-surface
rule in §5.7.1, and only when a deadline feature is present. A single vacancy — even a CFO — is a
lead to research, not a score to serve.

Specific composites worth testing as hypotheses:

- **`exec_exodus`** — two or more critical-role vacancies within six months, no backfill postings.
- **`frozen_org`** — open roles down >50% QoQ, no postings added in 60 days, 5500 participants down.
- **`licensed_contraction`** — MLO/rep/producer roster down >30%, or sponsoring entity terminated.
- **`maintainer_loss`** — bus factor 1 and no release in 180 days, or top-2 contributors silent.
- **`planned_transition`** — vacancy plus a same-role posting within 30 days. Low signal; exclude
  from the positive class so it does not dilute the rest.

Peer normalization is not optional. A 25% annual attrition number is normal in sales and alarming in
a five-person founding team. Compute `z_flow` within size band × stage × role class, from the same
source, or the feature measures the vendor's coverage rather than the company.

---

## 6. The same event can mean the opposite thing

| Observation | Context A | Meaning | Context B | Meaning |
| --- | --- | --- | --- | --- |
| Layoff | runway > 12 months, then a CFO or corp-dev hire | sale grooming (positive) | runway < 6 months | distress, already late |
| CFO departs | no successor, banker page or data-room cert active | process underway | successor named within 30 days | planned transition |
| Founder departs | no new venture announced | board-driven, sale likely | new company announced | talent exit, not a sale |
| Core maintainer stops | repo archived or licence changed | wind-down | new maintainer starts committing | handover |
| MLO roster shrinks | sponsor bank exited | licence/channel loss | no sponsor change | hiring freeze |
| Engineer churn rises | after an announced process | retention failure | before any process, with hiring freeze | genuine contraction |

These are hypotheses to test in Phase 0, not facts to encode. The layoff row is the one that
contradicts the README's "distress is confirmation" line: a layoff with a year of runway and a
strategic hire afterwards is often the company cleaning itself up for a sale.

---

## 7. Capture contract

Every people/headcount source must satisfy this before it enters the pipeline:

- **Completeness class** (A/B/C) recorded on the source registry row. Only A emits events.
- **`first_seen_at` / `last_seen_at` per (company, role_class)**, plus the raw observation and its
  URL. A roster slot is a time interval, not a row.
- **Person identity** comes from the source's own ID — GitHub login, inventor ID, NMLS ID, CRD,
  patent assignee — never from fuzzy name matching across sources. Do not build a person master.
- **Names are stored only where the source is a public record** (officer filings, licence rosters,
  patent inventors). Aggregated role counts elsewhere. No contact data, no résumés, no profiles.
- **Cadence**: daily for ATS, GitHub, CT and press; weekly for team/leadership pages (conditional
  GET); quarterly for LCA and patents; annual for 5500 and state reports; monthly for licensed
  vendor deltas.
- **Snapshot-first.** Class B and C surfaces have no usable history. Start recording them during
  Phase 0 even though they cannot contribute to the backtest; they are the only way they ever become
  backtestable.
- **PIT**: presence recorded at time T is usable at T; absence is usable only for class A. Never
  backfill a team page from today into a 2022 snapshot.

---

## 8. The skip list, item by item

Three different questions get conflated when someone says "just scrape LinkedIn":

- the **signal** — would this source tell us something about a departure or a headcount change?
- the **collection method** — scrape, licence, or read manually?
- the **unit** — a company/role state, or a named person?

We reject some collection methods without rejecting the signal. Below is the full position, because
"we don't do LinkedIn" is not an argument until it names the substitute.

| Source | Signal it would give | Why not automated | Lawful substitute | Verdict |
| --- | --- | --- | --- | --- |
| **LinkedIn** | employment spells, start/end dates, titles — the canonical departure record | User agreement prohibits automated collection; the CFAA ruling in *hiQ* did not settle the contract question, and access disputes have repeatedly drawn litigation; person-level PII at scale | Licensed workforce data (Revelio Labs, People Data Labs, Coresignal, Live Data) for spells; state filings and licence rosters for named officers | **Buy the signal, never scrape it.** Contract must permit derived signals and model training |
| **Glassdoor** | morale, layoff rumours, management changes | Self-selected reviews, sparse below ~50 employees, fake-review problem, no general-purpose API, ToS; reputational and defamation exposure if repeated | WARN notices, Layoffs.fyi, ATS deltas; an analyst may read a page manually | **Skip as a feature.** Manual read only, never a served claim |
| **Blind** | early layoff and reorg rumour, often before WARN | Anonymous and unverifiable, no representative sampling, no API, ToS; repeating a rumour is a defamation surface | WARN plus licensed employment data | **Skip.** Context for an analyst, never a detector |
| **Indeed** | aggregate job postings, hiring freeze | Duplicates ATS boards we already read, partner-only API, ToS; job postings are not departures | ATS JSON, HN "Who is hiring", USCIS H-1B hub; licensed postings feeds (LinkUp, Lightcast) if aggregate demand is ever needed | **Skip.** No unique signal for this universe |
| **Résumé databases** | employment history, titles, tenure | Purpose-limited to recruiting, PII, contract restrictions; using them for M&A surveillance is outside the licensed purpose | Licensed workforce data with an explicit purpose clause | **Skip** |
| **Social posts** (X, personal blogs, "exciting new chapter") | founder intent, departures | Individual intent inference, low precision, paid/limited API, ToS; the exposure is the claim, not the page | A manual 200-account watchlist that logs a URL into the verification queue (already allowed in the Phase 0 doc); explicit-intent sources for actual intent | **Manual only.** Never a feature, never a score |
| **Employee reviews** | same as Glassdoor | same as Glassdoor | same as Glassdoor | **Skip** |
| **Purchased email lists** | nothing about intent — this is contact data | Contact data is not a signal; CAN-SPAM, privacy law, list quality, deliverability; buying it and calling it intent is how data products get a bad reputation | Contact data as its own compliant lane, with consent, suppression and purpose separation | **Skip as a signal.** Keep outreach data in a separate lane |
| **Discord** | community admin changes, sentiment | Private or semi-private spaces, no representative sampling, ToS, no API | GitHub and package registries already cover devtools | **Skip** |

Two clarifications that matter more than the table:

- **The licensed workforce channel is the price of not scraping.** It is not a nice-to-have add-on;
  it is the only lawful way to see IC-level departures at universe scale. If the licence is too
  expensive, then `talent_exodus` in v1 is **leadership-only** — officer changes, 5500 participant
  declines, maintainer loss — and that scope decision should be written down rather than quietly
  assuming ATS boards cover people movement. They do not.
- **The manual exception is real and bounded.** An analyst may read any public page and log a lead
  with a URL; that lead enters `verification`, not the feature store. It never becomes a column, a
  training label or a score component. This is how the Phase 0 doc's 200-account founder watchlist
  stays legal while the crawler stays out of it.

What remains non-negotiable, independent of source:

- **No individual "flight risk" scoring.** Detecting that a role is vacant is a fact about a
  company; scoring a named person's likelihood of leaving is a different product with a different
  legal and ethical profile, and it is the one that gets the company written up badly.
- **No cross-source person resolution.** The accuracy is poor and the privacy cost is real.
- **No résumé or contact data mixed into this lane**, even when it arrives through a vendor that also
  sells workforce signals. Separate purpose, separate table, separate retention.


---

## 9. Phase placement

**Phase 0 (measure, do not build):**

1. Coverage audit on the 400 exits: what share has a state annual report with officers, a Form 5500,
   an LCA filing, a patent, a Wayback-captured team page at T−12? Drop any source below ~10%.
2. Standalone hazard ratios for: officer change, 5500 participant decline, LCA decline, patent
   inventor departure. One source at a time, before any composite.
3. Licensed-vendor trial on 100 exits: measure how many employment spells are recoverable at T−12
   and what the contract permits. Price it against the lift it adds.
4. Start daily ATS and weekly team-page snapshots on the Phase 0 universe from day one.

**Phase 1 (build the free layer):** ATS lifecycle, team/leadership diffs, GitHub org and archive,
state reports and 5500 ingestion, `role_vacancy` table, `planned_transition` exclusion.

**Phase 2 (buy the precise layer):** licensed workforce data at universe scale, maintainer graph and
patent flow, NMLS/CRD/producer rosters per vertical, analyst queue for sublease/auction/import.

**Prospective-only by construction** until they have a year of snapshots: ATS lifecycle, team pages,
licence rosters, GitHub membership. Do not let a Phase 0 lift table imply these were tested.

---

## 10. Open questions

- [ ] Which workforce vendor, and do its terms permit derived signals and model training?
- [ ] Does the state annual-report coverage justify per-state parsers, or only the top five states?
- [ ] Is a vacancy with no backfill strong enough to serve on its own, or only as a component?
- [ ] Do we store executive names at all, or only hashed role slots with a link to the filing?
- [ ] How do we measure the peer attrition baseline before the licensed feed is bought?
- [ ] Does `talent_exodus` belong in v1 at all, or is it a Phase 2 feature behind the paid layer?
